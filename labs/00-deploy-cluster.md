# **Creating an OpenShift Cluster**

REWORK STILL REQUIRED

In the previous lab we configured a disconnected registry and httpd cache for RHCOS images.  This will allow us to test our disconnected OpenShift deploy in this section of the lab.  But before we begin lets look at a few parts of the install-config.yaml configuration file.  In the file we can see there some interesting sections and attributes which we call out with the "**<===**" notation below:

~~~bash
[lab-user@provision scripts]$ cat install-config.yaml
apiVersion: v1
baseDomain: dynamic.opentlc.com
metadata:
  name: schmaustech <===CLUSTER NAME
networking:
  networkType: OpenShiftSDN <===NETWORK SDN TO USE ON DEPLOY
  machineCIDR: 10.20.0.0/24 <=== EXTERNAL/BAREMETAL NETWORK
compute:
- name: worker
  replicas: 2 <===NUMBER OF WORKERS ON DEPLOYMENT
controlPlane:
  name: master
  replicas: 3 <===NUMBER OF MASTERS ON DEPLOYMENT
  platform:
    baremetal: {}
platform:
  baremetal:
    provisioningNetworkCIDR: 172.22.0.0/24 <=== SUBNET OF PROVISIONING NETWORK
    provisioningNetworkInterface: ens3
    apiVIP: 10.20.0.110
    ingressVIP: 10.20.0.112
    dnsVIP: 10.20.0.111
    bootstrapOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz?sha256=c9e2698d0f3bcc48b7c66d7db901266abf27ebd7474b6719992de2d8db96995a
    clusterOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz?sha256=359e7c3560fdd91e64cd0d8df6a172722b10e777aef38673af6246f14838ab1a
    hosts:
      - name: master-0
        role: master
        bmc:
          address: ipmi://10.20.0.3:6204
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:40
        hardwareProfile: openstack
      - name: master-1
        role: master
        bmc:
          address: ipmi://10.20.0.3:6201
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:41
        hardwareProfile: openstack
      - name: master-2
        role: master
        bmc:
          address: ipmi://10.20.0.3:6200
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:42
        hardwareProfile: openstack
      - name: worker-0
        role: worker
        bmc:
          address: ipmi://10.20.0.3:6205
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:50
        hardwareProfile: openstack
      - name: worker-1
        role: worker
        bmc:
          address: ipmi://10.20.0.3:6202
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:51
        hardwareProfile: openstack
sshKey: 'ssh-rsa REDACTED SSH KEY lab-user@provision'
imageContentSources:
- mirrors:
  - provision.schmaustech.students.osp.opentlc.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - provision.schmaustech.students.osp.opentlc.com:5000/ocp4/openshift4
  source: registry.svc.ci.openshift.org/ocp/release
pullSecret: 'REDACTED PULL SECRET'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
 REDACTED CERTIFICATE
  -----END CERTIFICATE-----
~~~

Now that we have examined the install-config.yaml we are ready to proceed with the deployment.  However before starting the deploy we should always make sure that the master and worker baremetal nodes we are going to use are in a powered off state:

~~~bash
[lab-user@provision scripts]$ for i in 0 1 2 3 4 5
 do
 /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p620$i -Uadmin -Predhat chassis power off
 done
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
[lab-user@provision scripts]$
~~~

> **NOTE:** These commands may fail (`Unable to set Chassis Power Control to Down/Off`), if the load on the underlying infrastructure is too high.  If this happens, simply re-run the "script" until it succeeds for all nodes.

## Deploying OpenShift

We're going to install the cluster using two steps, one to **create the manifests** and one to **install the cluster**. Generating the manifests separately like this isn't necessary as it is done automatically when we run a `create cluster`.  However it's interesting to be able to see these files so we've done it to allow you a chance to explore! Additionally, if you had more configuration files this would be where you would add them.

### Create the manifests

Ok, let's create our cluster state directory:

~~~bash
[lab-user@provision scripts]$ mkdir $HOME/scripts/ocp
~~~

And place our install-config.yaml file into it:

~~~bash
[lab-user@provision scripts]$ cp $HOME/scripts/install-config.yaml $HOME/scripts/ocp
~~~

> **NOTE**: The installer will consume the install-config.yaml and remove the file from the state direcrtory. If you have not saved it somewhere else you can regenerate it with `openshift-baremetal-install create install-config --dir=ocp` on a running cluster.

~~~bash
[lab-user@provision scripts]$ $HOME/scripts/openshift-baremetal-install --dir=ocp --log-level debug create manifests
DEBUG OpenShift Installer 4.5.12                    
DEBUG Built from commit 9893a482f310ee72089872f1a4caea3dbec34f28 
DEBUG Fetching Master Machines...                  
DEBUG Loading Master Machines...                   
(...)
DEBUG   Loading Private Cluster Outbound Service... 
DEBUG   Loading Baremetal Config CR...             
DEBUG   Loading Image...                           
WARNING Discarding the Openshift Manifests that was provided in the target directory because its dependencies are dirty and it needs to be regenerated 
DEBUG   Fetching Install Config...                 
DEBUG   Reusing previously-fetched Install Config  
DEBUG   Fetching Cluster ID...                     
DEBUG   Reusing previously-fetched Cluster ID      
DEBUG   Fetching Kubeadmin Password...             
DEBUG   Generating Kubeadmin Password...           
DEBUG   Fetching OpenShift Install (Manifests)...  
DEBUG   Generating OpenShift Install (Manifests)... 
DEBUG   Fetching CloudCredsSecret...               
DEBUG   Generating CloudCredsSecret...             
DEBUG   Fetching KubeadminPasswordSecret...        
DEBUG   Generating KubeadminPasswordSecret...      
DEBUG   Fetching RoleCloudCredsSecretReader...     
DEBUG   Generating RoleCloudCredsSecretReader...   
DEBUG   Fetching Private Cluster Outbound Service... 
DEBUG   Generating Private Cluster Outbound Service... 
DEBUG   Fetching Baremetal Config CR...            
DEBUG   Generating Baremetal Config CR...          
DEBUG   Fetching Image...                          
DEBUG   Reusing previously-fetched Image           
DEBUG Generating Openshift Manifests...  
~~~

If we look in the manifests directory we can see there are all sorts of configuration items.  As mentioned, additonal configruation could be placed here for customizations of the cluster before actually kicking off the deploy:

~~~bash
[lab-user@provision scripts]$ ls -l $HOME/scripts/ocp/manifests/
total 116
-rw-r-----. 1 lab-user users  169 Oct 14 11:08 04-openshift-machine-config-operator.yaml
-rw-r-----. 1 lab-user users 6309 Oct 14 11:08 cluster-config.yaml
-rw-r-----. 1 lab-user users  154 Oct 14 11:08 cluster-dns-02-config.yml
-rw-r-----. 1 lab-user users  542 Oct 14 11:08 cluster-infrastructure-02-config.yml
-rw-r-----. 1 lab-user users  159 Oct 14 11:08 cluster-ingress-02-config.yml
-rw-r-----. 1 lab-user users  513 Oct 14 11:08 cluster-network-01-crd.yml
-rw-r-----. 1 lab-user users  272 Oct 14 11:08 cluster-network-02-config.yml
-rw-r-----. 1 lab-user users  142 Oct 14 11:08 cluster-proxy-01-config.yaml
-rw-r-----. 1 lab-user users  171 Oct 14 11:08 cluster-scheduler-02-config.yml
-rw-r-----. 1 lab-user users  264 Oct 14 11:08 cvo-overrides.yaml
-rw-r-----. 1 lab-user users 1335 Oct 14 11:08 etcd-ca-bundle-configmap.yaml
-rw-r-----. 1 lab-user users 3962 Oct 14 11:08 etcd-client-secret.yaml
-rw-r-----. 1 lab-user users  423 Oct 14 11:08 etcd-host-service-endpoints.yaml
-rw-r-----. 1 lab-user users  271 Oct 14 11:08 etcd-host-service.yaml
-rw-r-----. 1 lab-user users 4009 Oct 14 11:08 etcd-metric-client-secret.yaml
-rw-r-----. 1 lab-user users 1359 Oct 14 11:08 etcd-metric-serving-ca-configmap.yaml
-rw-r-----. 1 lab-user users 3921 Oct 14 11:08 etcd-metric-signer-secret.yaml
-rw-r-----. 1 lab-user users  156 Oct 14 11:08 etcd-namespace.yaml
-rw-r-----. 1 lab-user users  334 Oct 14 11:08 etcd-service.yaml
-rw-r-----. 1 lab-user users 1336 Oct 14 11:08 etcd-serving-ca-configmap.yaml
-rw-r-----. 1 lab-user users 3894 Oct 14 11:08 etcd-signer-secret.yaml
-rw-r-----. 1 lab-user users  301 Oct 14 11:08 image-content-source-policy-0.yaml
-rw-r-----. 1 lab-user users  296 Oct 14 11:08 image-content-source-policy-1.yaml
-rw-r-----. 1 lab-user users  118 Oct 14 11:08 kube-cloud-config.yaml
-rw-r-----. 1 lab-user users 1304 Oct 14 11:08 kube-system-configmap-root-ca.yaml
-rw-r-----. 1 lab-user users 4094 Oct 14 11:08 machine-config-server-tls-secret.yaml
-rw-r-----. 1 lab-user users 3993 Oct 14 11:08 openshift-config-secret-pull-secret.yaml
-rw-r-----. 1 lab-user users 2411 Oct 14 11:08 user-ca-bundle-config.yaml
~~~

### Create the cluster

We have now arrived at the point where we can run the `create cluster` argument for the install command to deploy our baremetal cluster.  This process will take about ~60-90 minutes to complete so have tmux running is you want to avoid network issues causing problems! :)

> **REMINDER**: Don't forget to use tmux!

~~~bash
[lab-user@provision scripts]$ $HOME/scripts/openshift-baremetal-install --dir=ocp --log-level debug create cluster
DEBUG OpenShift Installer 4.5.12                    
DEBUG Built from commit 9893a482f310ee72089872f1a4caea3dbec34f28 
DEBUG Fetching Metadata...                         
DEBUG Loading Metadata...                          
DEBUG   Loading Cluster ID...                      
DEBUG     Loading Install Config...                
DEBUG       Loading SSH Key...                     
DEBUG       Loading Base Domain...                 
DEBUG         Loading Platform...                  
DEBUG       Loading Cluster Name...                
DEBUG         Loading Base Domain...               
DEBUG         Loading Platform...                  
DEBUG       Loading Pull Secret...                 
DEBUG       Loading Platform...                    
DEBUG     Using Install Config loaded from state file 
DEBUG   Using Cluster ID loaded from state file
(...)
INFO Obtaining RHCOS image file from 'http://10.20.0.2/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz?sha256=c9e2698d0f3bcc48b7c66d7db901266abf27ebd7474b6719992de2d8db96995a' 
INFO The file was found in cache: /home/lab-user/.cache/openshift-installer/image_cache/ad57fdbef98553f778ac17b95b094a1a. Reusing... 
INFO Consuming OpenShift Install (Manifests) from target directory 
(...)
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [2m10s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[0]: Still creating... [2m10s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [2m10s elapsed] 
DEBUG module.bootstrap.libvirt_volume.bootstrap: Still creating... [2m10s elapsed] 
DEBUG module.bootstrap.libvirt_ignition.bootstrap: Still creating... [2m10s elapsed] 
DEBUG module.bootstrap.libvirt_volume.bootstrap: Creation complete after 2m17s [id=/var/lib/libvirt/images/schmaustech-mhnfj-bootstrap] 
DEBUG module.bootstrap.libvirt_ignition.bootstrap: Creation complete after 2m17s [id=/var/lib/libvirt/images/schmaustech-mhnfj-bootstrap.ign;5f7b64ba-cc8f-cec4-bf8c-47a4883c9bf6] 
DEBUG module.bootstrap.libvirt_domain.bootstrap: Creating... 
DEBUG module.bootstrap.libvirt_domain.bootstrap: Creation complete after 1s [id=008b263f-363d-4685-a2ca-e8852e3b5d05] 
(...)
DEBUG module.masters.ironic_node_v1.openshift-master-host[0]: Creation complete after 24m20s [id=63c12136-0605-4b0b-a2b3-b53b992b8189] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m21s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [24m21s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m31s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [24m31s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Still creating... [24m41s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m41s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[2]: Creation complete after 24m41s [id=a84a8327-3ecc-440c-91a2-fcf6546ab1f1] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [24m51s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Still creating... [25m1s elapsed] 
DEBUG module.masters.ironic_node_v1.openshift-master-host[1]: Creation complete after 25m2s [id=cb208530-0ff9-4946-bd11-b514190a56c1] 
DEBUG module.masters.data.ironic_introspection.openshift-master-introspection[0]: Refreshing state... 
DEBUG module.masters.data.ironic_introspection.openshift-master-introspection[2]: Refreshing state... 
DEBUG module.masters.data.ironic_introspection.openshift-master-introspection[1]: Refreshing state... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[0]: Creating... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[2]: Creating... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[1]: Creating... 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[1]: Creation complete after 2s [id=5920c1cc-4e14-4563-b9a4-12618ca315ba] 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[2]: Creation complete after 3s [id=9f371836-b9bd-4bd1-9e6d-d604c6c9d1b8] 
DEBUG module.masters.ironic_allocation_v1.openshift-master-allocation[0]: Creation complete after 3s [id=0537b2e4-8ba4-42a4-9f93-0a138444ae42] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Creating... 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[1]: Creating... 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[2]: Creating... 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Still creating... [10s elapsed] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[2]: Still creating... [10s elapsed] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[1]: Still creating... [10s elapsed] 
(...)
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Still creating... [9m0s elapsed] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Creation complete after 9m4s [id=63c12136-0605-4b0b-a2b3-b53b992b8189] 
DEBUG                                              
DEBUG Apply complete! Resources: 12 added, 0 changed, 0 destroyed. 
DEBUG OpenShift Installer 4.5.12                 
DEBUG Built from commit 0d5c871ce7d03f3d03ab4371dc39916a5415cf5c 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.schmaustech.dynamic.opentlc.com:6443... 
INFO API v1.18.3+6c42de8 up                       
INFO Waiting up to 40m0s for bootstrapping to complete...
(...)
DEBUG Bootstrap status: complete                   
INFO Destroying the bootstrap resources... 
(...)
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: downloading update 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: 0% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: 41% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: 57% complete 
(...)
DEBUG Still waiting for the cluster to initialize: Some cluster operators are still updating: authentication, console, csi-snapshot-controller, ingress, kube-storage-version-migrator, monitoring 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: 86% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: 86% complete 
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: 86% complete 
(...)
DEBUG Still waiting for the cluster to initialize: Working towards 4.5.12: 92% complete 
DEBUG Cluster is initialized                       
INFO Waiting up to 10m0s for the openshift-console route to be created... 
DEBUG Route found in openshift-console namespace: console 
DEBUG Route found in openshift-console namespace: downloads 
DEBUG OpenShift console route is created           
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/lab-user/scripts/ocp/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.schmaustech.dynamic.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "5VGM2-uMov3-4N2Vi-n5i3H" 
DEBUG Time elapsed per stage:                      
DEBUG     Infrastructure: 34m17s                   
DEBUG Bootstrap Complete: 35m56s                   
DEBUG  Bootstrap Destroy: 10s                      
DEBUG  Cluster Operators: 38m6s                    
INFO Time elapsed: 1h48m36s  
~~~

### Deployment Failure

> **Note**: Skip this section and go right to "Deployment Success" if your install succeed as indicated above!

If your deployment ends with any FATAL or ERROR lines such as:

~~~bash
DEBUG Still waiting for the cluster to initialize: Cluster operator console is reporting a failure: RouteHealthDegraded: failed to GET route (https://console-openshift-console.apps.vd44m.dynamic.opentlc.com/health): Get https://console-openshift-console.apps.vd44m.dynamic.opentlc.com/health: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
ERROR Cluster operator authentication Degraded is True with RouteHealth_FailedGet: RouteHealthDegraded: failed to GET route: EOF
INFO Cluster operator authentication Progressing is Unknown with NoData:
INFO Cluster operator authentication Available is Unknown with NoData:
ERROR Cluster operator console Degraded is True with RouteHealth_StatusError: RouteHealthDegraded: route not yet available, https://console-openshift-console.apps.vd44m.dynamic.opentlc.com/health returns
'503 Service Unavailable'
INFO Cluster operator console Progressing is True with SyncLoopRefresh_InProgress: SyncLoopRefreshProgressing: Working toward version 4.5.12
INFO Cluster operator console Available is False with Deployment_InsufficientReplicas: DeploymentAvailable: 0 pods available for console deployment
INFO Cluster operator insights Disabled is False with :
FATAL failed to initialize the cluster: Cluster operator console is reporting a failure: RouteHealthDegraded: failed to GET route (https://console-openshift-console.apps.vd44m.dynamic.opentlc.com/health): Get https://console-openshift-console.apps.vd44m.dynamic.opentlc.com/health: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
~~~

...you will need to carry out a few additional tasks to help complete the installation:

1. Check if the nodes are in a "**Ready**" state:

   ~~~bash
   [lab-user@provision scripts]$ oc --kubeconfig $HOME/scripts/ocp/auth/kubeconfig get nodes
   NAME                                 STATUS   ROLES    AGE   VERSION
   master-0.vd44m.dynamic.opentlc.com   Ready    master   81m   v1.18.3+47c0e71
   master-1.vd44m.dynamic.opentlc.com   Ready    master   81m   v1.18.3+47c0e71
   master-2.vd44m.dynamic.opentlc.com   Ready    master   81m   v1.18.3+47c0e71
   worker-0.vd44m.dynamic.opentlc.com   Ready    worker   40m   v1.18.3+47c0e71
   worker-1.vd44m.dynamic.opentlc.com   Ready    worker   39m   v1.18.3+47c0e71
   ~~~

   If any nodes report "**NotReady**" please run the specially prepared script at `/home/lab-user/scripts/fix-overlay.sh` on the provisioning host:

   ~~~bash
   [lab-user@provision scripts]$ sh $HOME/scripts/fix-overlay.sh
   (no output)
   ~~~

   > **NOTE**: If all of the nodes are in a "Ready State" a shown above you do **not** need to run this script, but it won't harm if you have done so - it will noop if all of your nodes are "Ready".

2. Kill the CoreDNS pods (they'll be automatically respawned, but we've seen DNS errors cause deployment failures) in the `openshift-kni-infra` namespace:

   ~~~bash
   [lab-user@provision scripts]$ for i in $(oc --kubeconfig $HOME/scripts/ocp/auth/kubeconfig get pods -A | awk '/coredns/ {print $2;}'); \
       do oc --kubeconfig $HOME/scripts/ocp/auth/kubeconfig delete pod $i -n openshift-kni-infra; done
   
   pod "coredns-master-0.xcs2v.dynamic.opentlc.com" deleted
   pod "coredns-master-1.xcs2v.dynamic.opentlc.com" deleted
   pod "coredns-master-2.xcs2v.dynamic.opentlc.com" deleted
   pod "coredns-worker-0.xcs2v.dynamic.opentlc.com" deleted
   pod "coredns-worker-1.xcs2v.dynamic.opentlc.com" deleted
   ~~~

3. Rerun the installer with the `wait-for install-complete` options:

   ~~~bash
   [lab-user@provision scripts]$ $HOME/scripts/openshift-baremetal-install \
       --dir=/home/lab-user/scripts/ocp --log-level debug wait-for install-complete
   ~~~

   This will reconnect to the installation process and allow you to again see the progress:

   ~~~bash
   DEBUG OpenShift Installer 4.5.12
   DEBUG Built from commit 9893a482f310ee72089872f1a4caea3dbec34f28
   DEBUG Fetching Install Config...
   DEBUG Loading Install Config...
   DEBUG   Loading SSH Key...
   DEBUG   Loading Base Domain...
   DEBUG     Loading Platform...
   DEBUG   Loading Cluster Name...
   DEBUG     Loading Base Domain...
   DEBUG     Loading Platform...
   DEBUG   Loading Pull Secret...
   DEBUG   Loading Platform...
   DEBUG Using Install Config loaded from state file
   DEBUG Reusing previously-fetched Install Config
   INFO Waiting up to 1h0m0s for the cluster at https://api.vd44m.dynamic.opentlc.com:6443 to initialize...
   ~~~

   Eventually (likely not more than 15-20 minutes, **but please contact a lab support person if you have issues and/or questions**) you'll get the cluster success and connection details as shown above. You can then move on to the "Deployment Success" section!

### Deployment Success

Once the cluster has successfully deployed at the end of the logging you will be presented with cluster command line information and also the login for the OpenShift console. Make sure to record those details somewhere convenient for later use. In the example above we seem them in these lines:

~~~bash 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.schmaustech.dynamic.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "5VGM2-uMov3-4N2Vi-n5i3H" 
~~~

Where the console is **https://console-openshift-console.apps.schmaustech.dynamic.opentlc.com** and the user to login is **kubeadmin** and the password is **5VGM2-uMov3-4N2Vi-n5i3H** (your's will be different than these of course). We can also interact with the cluster easily via the command line using the `oc` command. Before we run the `oc` commands we need to export the KUBECONFIG variable:

~~~bash
[lab-user@provision ~]$ export KUBECONFIG=$HOME/scripts/ocp/auth/kubeconfig
~~~

Now we can validate and confirm we have a 3 master and 2 worker cluster instantiated by issuing the `oc get nodes` command:

~~~bash
[lab-user@provision scripts]$ oc get nodes
NAME                                 STATUS   ROLES    AGE   VERSION
master-0.xcs2v.dynamic.opentlc.com   Ready    master   79m   v1.18.3+47c0e71
master-1.xcs2v.dynamic.opentlc.com   Ready    master   79m   v1.18.3+47c0e71
master-2.xcs2v.dynamic.opentlc.com   Ready    master   79m   v1.18.3+47c0e71
worker-0.xcs2v.dynamic.opentlc.com   Ready    worker   57m   v1.18.3+47c0e71
worker-1.xcs2v.dynamic.opentlc.com   Ready    worker   58m   v1.18.3+47c0e71

~~~

Further we can also confirm all the cluster operators are functional and available by looking at the `oc get clusteroperators` command:

~~~bash
[lab-user@provision scripts]$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.12    True        False         False      3m15s
cloud-credential                           4.5.12    True        False         False      52m
cluster-autoscaler                         4.5.12    True        False         False      31m
config-operator                            4.5.12    True        False         False      31m
console                                    4.5.12    True        False         False      8m45s
csi-snapshot-controller                    4.5.12    True        False         False      14m
dns                                        4.5.12    True        False         False      38m
etcd                                       4.5.12    True        False         False      38m
image-registry                             4.5.12    True        False         False      55s
ingress                                    4.5.12    True        False         False      15m
insights                                   4.5.12    True        False         False      34m
kube-apiserver                             4.5.12    True        False         False      37m
kube-controller-manager                    4.5.12    True        False         False      38m
kube-scheduler                             4.5.12    True        False         False      37m
kube-storage-version-migrator              4.5.12    True        False         False      15m
machine-api                                4.5.12    True        False         False      26m
machine-approver                           4.5.12    True        False         False      37m
machine-config                             4.5.12    True        False         False      39m
marketplace                                4.5.12    True        False         False      12m
monitoring                                 4.5.12    True        False         False      14m
network                                    4.5.12    True        False         False      39m
node-tuning                                4.5.12    True        False         False      39m
openshift-apiserver                        4.5.12    True        False         False      15m
openshift-controller-manager               4.5.12    True        False         False      32m
openshift-samples                          4.5.12    True        False         False      15m
operator-lifecycle-manager                 4.5.12    True        False         False      39m
operator-lifecycle-manager-catalog         4.5.12    True        False         False      39m
operator-lifecycle-manager-packageserver   4.5.12    True        False         False      15m
service-ca                                 4.5.12    True        False         False      39m
storage                                    4.5.12    True        False         False      34m

~~~

Finally, we need to patch the Image Registry Operator.  In a production environment the Image Registry Operator needs to be configured with shared storage; however, since this is a lab we can just configure it with an empty directory:

~~~bash
[lab-user@provision ~]$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
	--patch '{"spec":{"storage":{"emptyDir":{}}}}'
config.imageregistry.operator.openshift.io/cluster patched

[lab-user@provision ~]$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
	--patch '{"spec":{"managementState":"Managed"}}'
config.imageregistry.operator.openshift.io/cluster patched
~~~

Once the above is configured we should be able to see the pods related to the Image Registry Operator:

~~~bash
[lab-user@provision ~]$ oc get pod -n openshift-image-registry
NAME                                               READY   STATUS      RESTARTS   AGE
cluster-image-registry-operator-574467db97-rkzb4   2/2     Running     0          18h
image-pruner-1601942400-wbkxx                      0/1     Completed   0          14h
image-registry-6fbc9d5597-lkt79                    1/1     Running     0          18m
node-ca-255lq                                      1/1     Running     0          18h
node-ca-2hd5t                                      1/1     Running     0          18h
node-ca-8wc2g                                      1/1     Running     0          18h
node-ca-ktzl6                                      1/1     Running     0          18h
node-ca-vmcq2                                      1/1     Running     0          18h
~~~

At this point you are now ready to move onto the next lab where we will look at the Machine Config Operator (aka Baremetal Operator). [Continue to the Baremetal Operator lab!](https://github.com/RHFieldProductManagement/baremetal-ipi-lab/blob/master/05-baremetal.md)

