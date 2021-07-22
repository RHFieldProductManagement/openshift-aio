# **Deploy OpenShift Virtualization**

This lab will focus on installing and configuring OpenShift Virtualization to enable the AIO OpenShift cluster to run virtual machines alongside containerised applications. The AIO deployment does allow for Containerized Virtualization deployment during the Ansible playbook run however the documentation here allows one to manually experience the installation process.  The mechanism for installation is to utilise the operator model via the cli, just like we did with the storage lab. Note, it's entirely possible to deploy via the web UI should you wish to do so, but we're not documenting that mechanism here.

The first step for installing Containerized Virtualization is to prepare the a yaml that will configure the namespace, operator group and subscription to install the operator.  We can accomplish that by creating the file below:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-cnv-operator-install.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
    - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  startingCSV: kubevirt-hyperconverged-operator.v2.6.5
  channel: "stable"
EOF
~~~

If we take a quick look at the file again we can distinctly see that the first section is about creating the namespace openshift-cnv, the second section is about creating the kubevirt-hyperconverged-group operator group and the last portion is for defining the subscription which ultimately installs the operator.

~~~bash
[root@ocp4-bastion ~]# cat  ~/openshift-cnv-operator-install.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
    - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  startingCSV: kubevirt-hyperconverged-operator.v2.6.5
  channel: "stable"
~~~

Lets go ahead and create the resources using the yaml we defined above:

~~~bash
[root@ocp4-bastion ~]# oc create -f ~/openshift-cnv-operator-install.yaml 
namespace/openshift-cnv created
operatorgroup.operators.coreos.com/kubevirt-hyperconverged-group created
subscription.operators.coreos.com/hco-operatorhub created
~~~

We can see this generates multiple operator pods related to Containerized Virtualization:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-cnv
NAME                                               READY   STATUS              RESTARTS   AGE
cdi-operator-7565b8bc7d-8bbmx                      0/1     ContainerCreating   0          6s
cluster-network-addons-operator-55464d8f45-tlcjw   0/1     ContainerCreating   0          8s
hco-operator-7b8c96b8b7-q7pl2                      0/1     ContainerCreating   0          9s
hco-webhook-5c9854d5cb-qtcqm                       0/1     ContainerCreating   0          8s
hostpath-provisioner-operator-695bf6fbd6-kl6cs     0/1     ContainerCreating   0          6s
node-maintenance-operator-5485f89b66-z9n4s         0/1     ContainerCreating   0          6s
ssp-operator-58fcc47b69-njlrx                      0/1     ContainerCreating   0          7s
vm-import-operator-87df7f45b-b9wm5                 0/1     ContainerCreating   0          6s
~~~

If we add a watch to our oc statement we can watch as each pod gets to a running state:

~~~bash
[root@ocp4-bastion ~]# watch -n2 'oc get pods -n openshift-cnv'
(...)
~~~

> **NOTE**: It may take a few minutes for the pods to start up properly. Press **Ctrl+C** to exit the watch command.

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-cnv
NAME                                               READY   STATUS    RESTARTS   AGE
cdi-operator-7565b8bc7d-8bbmx                      1/1     Running   0          110s
cluster-network-addons-operator-55464d8f45-tlcjw   1/1     Running   0          112s
hco-operator-7b8c96b8b7-q7pl2                      1/1     Running   0          113s
hco-webhook-5c9854d5cb-qtcqm                       1/1     Running   0          112s
hostpath-provisioner-operator-695bf6fbd6-kl6cs     1/1     Running   0          110s
node-maintenance-operator-5485f89b66-z9n4s         1/1     Running   0          110s
ssp-operator-58fcc47b69-njlrx                      1/1     Running   0          111s
virt-operator-768746dd9c-mknb7                     0/1     Running   0          29s
virt-operator-768746dd9c-pm4jt                     0/1     Running   0          29s
vm-import-operator-87df7f45b-b9wm5                 1/1     Running   0          110s
~~~

At this point we are now ready to to deploy a HyperConverged configuration which assumes OCS/ODF was deployed in the previous lab or by default with the AIO playbook as a storage source for the virtual machines.  If this is not the case, NFS or hostpath provisioner, which is a local storage provisioner designed for OpenShift Virtualization, must be enabled first.  The instructions for that can be found [here](https://github.com/RHFieldProductManagement/openshift-aio/blob/main/labs/00-deploy-hostpath-cnv.md)

To configure a HyperConverged deployment we first need to create the following resource yaml:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-cnv-hyperconverged.yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
EOF
~~~

Next lets apply this resource yaml to our cluster:

~~~bash
[root@ocp4-bastion ~]# oc create -f openshift-cnv-hyperconverged.yaml
hyperconverged.hco.kubevirt.io/kubevirt-hyperconverged created
~~~

During this process you will see a lot of pods create and terminate, which will look something like the following depending on when you view it; it's always changing:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-cnv
NAME                                                  READY   STATUS    RESTARTS   AGE
bridge-marker-chrz8                                   1/1     Running   0          7m11s
bridge-marker-gf2bl                                   1/1     Running   0          7m10s
bridge-marker-grhfx                                   1/1     Running   0          7m10s
bridge-marker-j4nqm                                   1/1     Running   0          7m10s
bridge-marker-r8qgj                                   1/1     Running   0          7m10s
bridge-marker-zwphx                                   1/1     Running   0          7m10s
cdi-apiserver-778c8749d5-tnxq7                        1/1     Running   0          7m8s
cdi-deployment-cf8bf58b9-8j65d                        1/1     Running   0          7m6s
cdi-operator-7565b8bc7d-bd2rh                         1/1     Running   0          9m40s
cdi-uploadproxy-5cd7794458-whpng                      1/1     Running   0          7m5s
(...)
~~~

This will continue for some time, depending on your environment. You will know the process is complete when you see in the CLI that the operator installation has been successful by running the following command and getting the "**Succeeded**" output:

~~~bash
[root@ocp4-bastion ~]# oc get csv -n openshift-cnv
NAME                                      DISPLAY                    VERSION   REPLACES                                  PHASE
kubevirt-hyperconverged-operator.v2.6.5   OpenShift Virtualization   2.6.5     kubevirt-hyperconverged-operator.v2.6.4   Succeeded
~~~

If you do not see `Succeeded` in the `PHASE` column then the deployment may still be progressing, or has failed. You will not be able to proceed until the installation has been successful. Once the `PHASE` changes to `Succeeded` you can validate that the required resources and the additional components have been deployed across the nodes. First let's check the pods deployed in the `openshift-cnv` namespace:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-cnv
NAME                                                  READY   STATUS    RESTARTS   AGE
bridge-marker-chrz8                                   1/1     Running   0          7m11s
bridge-marker-gf2bl                                   1/1     Running   0          7m10s
bridge-marker-grhfx                                   1/1     Running   0          7m10s
bridge-marker-j4nqm                                   1/1     Running   0          7m10s
bridge-marker-r8qgj                                   1/1     Running   0          7m10s
bridge-marker-zwphx                                   1/1     Running   0          7m10s
cdi-apiserver-778c8749d5-tnxq7                        1/1     Running   0          7m8s
cdi-deployment-cf8bf58b9-8j65d                        1/1     Running   0          7m6s
cdi-operator-7565b8bc7d-bd2rh                         1/1     Running   0          9m40s
cdi-uploadproxy-5cd7794458-whpng                      1/1     Running   0          7m5s
(...)
~~~

> **NOTE**: All pods shown from this command should be in the `Running` state. You will have many different types, the above snippet is just an example of the output at one point in time, you may have more or less at any one point. Below we discuss some of the pod types and what they do.

Together, all of these pods are responsible for various functions of running a virtual machine on-top of OpenShift/Kubernetes. See the table below that describes some of the various different pod types and their function:

Pod Name                                                                                           | Pod Responsibilities
-------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
_[bridge-marker](https://github.com/kubevirt/bridge-marker)_                                       | Marks network bridges as available node resources.
_[cdi-_](<https://github.com/kubevirt/containerized-data-importer)*>                               | The Containerised Data Importer (CDI) is a Kubernetes extension to populate PVCs with VM disk images or other data. CDI pods allow OpenShift virtualisation to import, upload and clone Virtual Machine images.
_[cluster-network-addons-operator](https://github.com/kubevirt/cluster-network-addons-operator)_   | Allows the installation of additional networking plugins.
_[hco-operator](https://github.com/kubevirt/hyperconverged-cluster-operator)_                      | Allows users to deploy and configure multiple operators in a single operator and via a single entry point. An "operator of operators."
_[hostpath-provisioner-operator](https://github.com/kubevirt/hostpath-provisioner-operator)_       | Operator that manages the hostpath-provisioner, which provisions storage on network filesystems mounted on the host.
_[kube-cni-linux-bridge-plugin](https://github.com/containernetworking/plugins)_                   | CNI Plugin to create a network bridge and add a host and container to it.
_kubemacpool-mac-controller-manager_                                                               | Allocation of MAC addresses from a pool to secondary interfaces.
_[kubevirt-node-labeller](https://github.com/kubevirt/node-labeller)_                              | Creates node labels based on CPU information.
_[kubevirt-ssp-operator](https://github.com/MarSik/kubevirt-ssp-operator)_                         | Scheduling, Scale and Performance operator for OpenShift. The Hyperconverged Cluster Operator automatically installs the SSP operator when deploying.
_nmstate-handler_                                                                                  | Deploys NMState which allows network administrators to manage host networking settings in a declarative manner.
_[node-maintenance-operator](https://github.com/kubevirt/cluster-network-addons-operator#nmstate)_ | Operator that allows the administrator to deploy the NMState State Controller.
_[ovs-cni](https://github.com/kubevirt/ovs-cni)_                                                   | The Open vSwitch CNI plugin.
_[virt-api](https://github.com/kubevirt/kubevirt/tree/master/pkg/virt-api)_                        | Kubernetes Virtualization API and runtime in order to define and manage virtual machines
_[virt-controller](https://kubernetes.io/blog/2018/05/22/getting-to-know-kubevirt/)_               | The operator that's responsible for cluster-wide virtualisation functionality
_[virt-handler](https://kubernetes.io/blog/2018/05/22/getting-to-know-kubevirt/)_                  | Tracks changes to a VM's state.
_[virt-template-validator](https://kubernetes.io/blog/2018/05/22/getting-to-know-kubevirt/)_       | Add-on to check the annotations on templates and reject them if invalid.

There's also a few custom resources that get defined too, for example the `NodeNetworkState` (`nns` for short) definitions that can be used with the `nmstate-handler` pods to ensure that the NetworkManager state on each node is configured as required, e.g. for defining interfaces/bridges on each of the machines for connectivity for both the physical machine itself and for providing network access for pods (and virtual machines) within OpenShift/Kubernetes:

```bash
[lab-user@provision ~]$ oc get nns -A
NAME                                 AGE
master-0.nlm9j.dynamic.opentlc.com   117s
master-1.nlm9j.dynamic.opentlc.com   2m
master-2.nlm9j.dynamic.opentlc.com   2m8s
worker-0.nlm9j.dynamic.opentlc.com   2m14s
worker-1.nlm9j.dynamic.opentlc.com   2m11s
worker-2.nlm9j.dynamic.opentlc.com   2m15s

[lab-user@provision ~]$ oc get nns/worker-2.$GUID.dynamic.opentlc.com -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkState
metadata:
  creationTimestamp: "2020-09-25T18:54:17Z"
  generation: 1
(...)
  name: worker-2
(...)
   interfaces:
- ipv4:
        enabled: false
      ipv6:
        enabled: false
      mac-address: 92:0a:eb:3f:f2:4e
      mtu: 1400
      name: br-ext
      state: down
      type: ovs-interface
    - ipv4:
        address:
        - ip: 172.22.0.48
          prefix-length: 24
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        dhcp: true
        enabled: true
      ipv6:
        address:
        - ip: fe80::5f23:c69d:ee2b:88a2
          prefix-length: 64
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        autoconf: true
        dhcp: true
        enabled: true
      mac-address: DE:AD:BE:EF:00:52
      mtu: 1500
      name: ens4
      state: up
      type: ethernet

(...)
```

Here you can see the current state of the node (some of the output has been cut), the interfaces attached, and their physical/logical addresses. Before we move on from this lab we need to ensure that we have setup a proper bridge for our VMs to get direct access to the external network that our workers are on, making it easy for us to connect to any VM's we decide to create. We can do this by creating a **NodeNetworkConfigurationPolicy** (nncp):

```bash
[lab-user@provision ocp]$ cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: worker-brext-ens3
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: brext
        description: brext with ens3
        type: linux-bridge
        state: up
        ipv4:
          enabled: true
          dhcp: true
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: ens3
EOF
nodenetworkconfigurationpolicy.nmstate.io/worker-brext-ens3 created
```

The above policy will attach the "brext" bridge to the external network interface **ens3**. We can watch the progress by requesting the **NodeNetworkConfigurationEnactment** (nnce) by running the following a few times:

```bash
[lab-user@provision ocp]$ oc get nnce
NAME                                                   STATUS
master-0.hhnfk.dynamic.opentlc.com.worker-brext-ens3   NodeSelectorNotMatching
master-1.hhnfk.dynamic.opentlc.com.worker-brext-ens3   NodeSelectorNotMatching
master-2.hhnfk.dynamic.opentlc.com.worker-brext-ens3   NodeSelectorNotMatching
worker-0.hhnfk.dynamic.opentlc.com.worker-brext-ens3   ConfigurationProgressing
worker-1.hhnfk.dynamic.opentlc.com.worker-brext-ens3   ConfigurationProgressing
worker-2.hhnfk.dynamic.opentlc.com.worker-brext-ens3   ConfigurationProgressing

[lab-user@provision ocp]$ oc get nnce
NAME                                                   STATUS
master-0.hhnfk.dynamic.opentlc.com.worker-brext-ens3   NodeSelectorNotMatching
master-1.hhnfk.dynamic.opentlc.com.worker-brext-ens3   NodeSelectorNotMatching
master-2.hhnfk.dynamic.opentlc.com.worker-brext-ens3   NodeSelectorNotMatching
worker-0.hhnfk.dynamic.opentlc.com.worker-brext-ens3   SuccessfullyConfigured
worker-1.hhnfk.dynamic.opentlc.com.worker-brext-ens3   SuccessfullyConfigured
worker-2.hhnfk.dynamic.opentlc.com.worker-brext-ens3   SuccessfullyConfigured
```

> **NOTE**: You may need to run the `oc get nnce` a fair few times before all three of your workers are "SuccessfullyConfigured".



If we debug one of the worker nodes we can see that a "brext" interface has been created:

```bash
[lab-user@provision ~]$ oc debug node/worker-0.$GUID.dynamic.opentlc.com
Starting pod/worker-0xcs2vdynamicopentlccom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.20.0.200
If you don't see a command prompt, try pressing enter.
sh-4.2# chroot /host

sh-4.4# ip a show brext
49: brext: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8942 qdisc noqueue state UP group default qlen 1000
    link/ether ba:dc:0f:fe:e0:50 brd ff:ff:ff:ff:ff:ff
    inet 172.22.0.48/24 brd 10.20.0.255 scope global dynamic noprefixroute brext
       valid_lft 43118sec preferred_lft 43118sec

sh-4.4# exit
exit
sh-4.2# exit
exit

Removing debug pod ...
```
>  **NOTE**: Make sure to exit out of the worker's debug pod before proceeding.

Once the bridges have been successfully configured we can then add a **NetworkAttachmentDefinition** which will tell OpenShift about this new bridge that can be consumed for virtual machines (note the "cnv-bridge" type):

```bash
[lab-user@provision ocp]$ oc project default
Now using project "default" on server "https://api.xcs2v.dynamic.opentlc.com:6443".

[lab-user@provision ocp]$ cat << EOF | oc apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: brext
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/brext
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "brext",
    "plugins": [
      {
        "type": "cnv-bridge",
        "bridge": "brext"
      },
      {
        "type": "tuning"
      }
    ]
  }'
EOF

networkattachmentdefinition.k8s.cni.cncf.io/tuning-bridge-fixed created
```

Once those have been applied we can now move forward in the lab.

Ready to test out this cool new environment and deploy some VM workloads onto your new cluster? Well, that's just what we are going to do in the next lab: [Running Workloads in the Environment](https://github.com/RHFieldProductManagement/openshift-aio/blob/main/labs/06-deploy-workloads-cnv.md).
