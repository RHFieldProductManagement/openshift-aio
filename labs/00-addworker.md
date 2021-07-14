# Add Additional Third Worker Node

When using the AIO provisioned lab environment it automatically creates 3 nodes that can be used as worker nodes.  However during the deployment of OpenShift with the AIO one can specify 0 workers and up to 3 workers.   In this lab we will show how to consume an additional worker if in the AIO my_vars.yaml less then 3 worker nodes were specified.

First lets confirm how many worker nodes were defined in the initial deployment:

~~~bash
[root@ocp4-bastion ~]# grep -A3 compute ~/lab/install-config.yaml
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 2
~~~

We can see from the output above that the AIO deployment used 2 of the possible 3 worker nodes available.  In that case lets proceed in adding the third available worker node into our OpenShift cluster.

First we need to supply a baremetal node defintion to the baremetal operator to do this.  Lets create the following baremetal node definition file; this has been pre-populated for you based on the pre-configured worker node in your dedicated environment:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/bmh.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: worker3-bmc-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cmVkaGF0
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: worker3
spec:
  online: true
  bootMACAddress: de:ad:be:ef:00:06
  bmc:
    address: ipmi://192.168.123.1:6236
    credentialsName: worker3-bmc-secret
  rootDeviceHints:
    deviceName: /dev/vda
EOF
~~~

Let's look at the file that it created:

~~~bash
[root@ocp4-bastion ~]# cat ~/bmh.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: worker3-bmc-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cmVkaGF0
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: worker3
spec:
  online: true
  bootMACAddress: de:ad:be:ef:00:06
  bmc:
    address: ipmi://192.168.123.1:6236
    credentialsName: worker3-bmc-secret
  rootDeviceHints:
    deviceName: /dev/vda
~~~

You'll see that this is set to create two different resources, one is a `secret` that contains a base64 encoded username/password for IPMI access for the baremetal host, and then a `BareMetalHost` object (linked to the secret) that will define the node within the cluster itself, which is enough for the baremetal operator to kick off the deployment.

Let's now create these resources:

~~~bash
[root@ocp4-bastion ~]# oc create -f ~/bmh.yaml -n openshift-machine-api
secret/worker3-bmc-secret created
baremetalhost.metal3.io/worker3 created
~~~

Immediately you should this being reflected in as a new `BareMetalHost`, note the new entry with a status of "**inspecting**":

~~~bash
[root@ocp4-bastion ~]# oc get baremetalhosts -n openshift-machine-api
NAME      STATUS   PROVISIONING STATUS      CONSUMER                   BMC                         HARDWARE PROFILE   ONLINE   ERROR
master1   OK       externally provisioned   aio-r4cj9-master-0         ipmi://192.168.123.1:6231                      true     
master2   OK       externally provisioned   aio-r4cj9-master-1         ipmi://192.168.123.1:6232                      true     
master3   OK       externally provisioned   aio-r4cj9-master-2         ipmi://192.168.123.1:6233                      true     
worker1   OK       provisioned              aio-r4cj9-worker-0-cxnxz   ipmi://192.168.123.1:6234   unknown            true     
worker2   OK       provisioned              aio-r4cj9-worker-0-7kr9w   ipmi://192.168.123.1:6235   unknown            true     
worker3   OK       inspecting                                          ipmi://192.168.123.1:6236                      true    
~~~

This additional worker node will now be inspected, data will be gathered about the system such as number of NIC's, storage disks, CPU, memory, and so on. The system will then sit in a holding pattern - all we've done is add it as a baremetal host, NOT an OpenShift worker, nor has it actually had any components installed on it, it's merely just running from a basic ramdisk for now.

Try watching the list of `BareMetalHosts` every 10s or so to see when it has finished this inspection step (you're looking for the node state of the worker to be '**ready**'):

~~~bash
[root@ocp4-bastion ~]# watch -n10 oc get baremetalhosts -n openshift-machine-api
(...)

worker3   OK       ready                                               ipmi://192.168.123.1:6236   unknown          true

(Ctrl-C to stop)
~~~

Now you should see the machine as '**ready**' in the list (you likely saw it in the 'watch' as above, but it's worth confirming):

~~~bash
[root@ocp4-bastion ~]# oc get baremetalhosts/worker3 -n openshift-machine-api
NAME      STATUS   PROVISIONING STATUS   CONSUMER   BMC                         HARDWARE PROFILE   ONLINE   ERROR
worker3   OK       ready                            ipmi://192.168.123.1:6236   unknown            true  
~~~

At this stage, this third worker node is just a baremetal host, it is not ready for running workloads, nor will it show up as an OpenShift node. For it to become managed by OpenShift as a `Machine` and for it to become a `Node`, so we can run applications on it, we need to scale the respective `MachineSet` up. This will kick off the process for deploying RHCOS, installing the OpenShift components, configuring the components to talk to our cluster, and ensuring everything is running properly. When the baremetal opertator is deployed, it creates a `MachineSet` for worker nodes automatically for us:

~~~bash
[root@ocp4-bastion ~]# oc -n openshift-machine-api get machineset
NAME                 DESIRED   CURRENT   READY   AVAILABLE   AGE
aio-r4cj9-worker-0   2         2         2       2           5d3h
~~~

Much like scaling other platforms that are integrated with OpenShift 4.x (e.g. AWS), to dynamically scale our cluster from available resources we simply need to increase the number of 'replicas' to our `MachineSet`. The primary difference between scaling a baremetal cluster and a cluster that utilises the public cloud is that you have to define all of your nodes in baremetal whilst with the public cloud it can spawn new VM's on-demand through the public cloud API.

Let's now scale our cluster:

~~~bash
[root@ocp4-bastion ~]# oc -n openshift-machine-api scale machineset aio-r4cj9-worker-0 --replicas=3
machineset.machine.openshift.io/aio-r4cj9-worker-0 scaled
~~~

Now if you check the `BareMetalHosts` you should see a slightly different state for our worker (ie provisioning):

~~~bash
[root@ocp4-bastion ~]# oc get baremetalhosts -n openshift-machine-api
NAME      STATUS   PROVISIONING STATUS      CONSUMER                   BMC                         HARDWARE PROFILE   ONLINE   ERROR
master1   OK       externally provisioned   aio-r4cj9-master-0         ipmi://192.168.123.1:6231                      true     
master2   OK       externally provisioned   aio-r4cj9-master-1         ipmi://192.168.123.1:6232                      true     
master3   OK       externally provisioned   aio-r4cj9-master-2         ipmi://192.168.123.1:6233                      true     
worker1   OK       provisioned              aio-r4cj9-worker-0-cxnxz   ipmi://192.168.123.1:6234   unknown            true     
worker2   OK       provisioned              aio-r4cj9-worker-0-7kr9w   ipmi://192.168.123.1:6235   unknown            true     
worker3   OK       provisioning             aio-r4cj9-worker-0-4t69v   ipmi://192.168.123.1:6236   unknown            true  
~~~

If you re-run an extended `watch` command you can keep track of the process across multiple components (you may need to enlarge your terminal window to see all of this output cleanly):

~~~bash
[root@ocp4-bastion ~]# watch -n10 "oc get baremetalhosts -n openshift-machine-api && echo && oc get nodes"

NAME	  STATUS   PROVISIONING STATUS      CONSUMER                   BMC                         HARDWARE PROFILE   ONLINE   ERROR
master1   OK	   externally provisioned   aio-r4cj9-master-0         ipmi://192.168.123.1:6231                      true
master2   OK	   externally provisioned   aio-r4cj9-master-1         ipmi://192.168.123.1:6232                      true
master3   OK	   externally provisioned   aio-r4cj9-master-2         ipmi://192.168.123.1:6233                      true
worker1   OK	   provisioned              aio-r4cj9-worker-0-cxnxz   ipmi://192.168.123.1:6234   unknown            true
worker2   OK	   provisioned              aio-r4cj9-worker-0-7kr9w   ipmi://192.168.123.1:6235   unknown            true
worker3   OK	   provisioning             aio-r4cj9-worker-0-4t69v   ipmi://192.168.123.1:6236   unknown            true

NAME                           STATUS   ROLES    AGE    VERSION
ocp4-master1.aio.example.com   Ready    master   5d3h   v1.20.0+87cc9a4
ocp4-master2.aio.example.com   Ready    master   5d3h   v1.20.0+87cc9a4
ocp4-master3.aio.example.com   Ready    master   5d3h   v1.20.0+87cc9a4
ocp4-worker1.aio.example.com   Ready    worker   5d3h   v1.20.0+87cc9a4
ocp4-worker2.aio.example.com   Ready    worker   5d3h   v1.20.0+87cc9a4


(Ctrl-C to stop)
~~~

> **NOTE**: This process should take around 10 minutes before the new "baremetal" machine shows up as a new node in the list. You may also want to enlarge your terminal window so everything displays clearly.

Once this third node appears in the nodes window, Ctrl-C the watch command, and let's verify the node status:

~~~bash
[root@ocp4-bastion ~]# oc get nodes
NAME                           STATUS     ROLES    AGE    VERSION
ocp4-master1.aio.example.com   Ready      master   5d3h   v1.20.0+87cc9a4
ocp4-master2.aio.example.com   Ready      master   5d3h   v1.20.0+87cc9a4
ocp4-master3.aio.example.com   Ready      master   5d3h   v1.20.0+87cc9a4
ocp4-worker1.aio.example.com   Ready      worker   5d3h   v1.20.0+87cc9a4
ocp4-worker2.aio.example.com   Ready      worker   5d3h   v1.20.0+87cc9a4
ocp4-worker3.aio.example.com   NotReady   worker   6s     v1.20.0+87cc9a4
~~~

It may demonstrate a '**NotReady**' status, if it does it will likely be because all of the necessary configuration will not have yet been applied, nor will it have all of the pods/containers running yet. Give it some time and all node should report in a '**Ready**' state.

~~~bash
[root@ocp4-bastion ~]# oc get nodes
NAME                           STATUS   ROLES    AGE     VERSION
ocp4-master1.aio.example.com   Ready    master   5d3h    v1.20.0+87cc9a4
ocp4-master2.aio.example.com   Ready    master   5d3h    v1.20.0+87cc9a4
ocp4-master3.aio.example.com   Ready    master   5d3h    v1.20.0+87cc9a4
ocp4-worker1.aio.example.com   Ready    worker   5d3h    v1.20.0+87cc9a4
ocp4-worker2.aio.example.com   Ready    worker   5d3h    v1.20.0+87cc9a4
ocp4-worker3.aio.example.com   Ready    worker   3m51s   v1.20.0+87cc9a4
~~~

That's it! We've successfully scaled our "baremetal" cluster using the baremetal operator.

Now that we have this worker, we can go ahead and use it. [Move on to the next lab to Deploy OpenShift Container Storage on to your cluster](https://github.com/RHFieldProductManagement/baremetal-ipi-lab/blob/master/07-deployocs.md).
