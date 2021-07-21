# **Creating an OpenShift Cluster**

The AIO environment can deploy the OpenShift cluster automatically if one is not interested in understanding how the deployment process works.  However in this lab we walk through the process of taking the disconnected registry created in the previous lab and the install-config.yaml that the AIO generated along with the modifications we have been making in the previous lab.

Before we begin our dpeloyment lets look at a few parts of the install-config.yaml configuration file.  In the file we can see there some interesting sections and attributes which we call out with the "**<===**" notation below:

~~~bash
[root@ocp4-bastion ~]# cat ~/lab/install-config.yaml 
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 2  <===NUMBER OF WORKERS ON DEPLOYMENT
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3  <===NUMBER OF MASTERS ON DEPLOYMENT
  platform:
    baremetal: {}
metadata:
  name: aio <===CLUSTER NAME
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN  <===NETWORK SDN TO USE ON DEPLOY
  serviceNetwork:
  - 172.30.0.0/16
  machineCIDR: 192.168.123.0/24  <=== EXTERNAL/BAREMETAL NETWORK
platform:
  baremetal:
    provisioningNetworkCIDR: 172.22.0.0/24  <=== SUBNET OF PROVISIONING NETWORK
    provisioningNetworkInterface: enp1s0
    apiVIP: 192.168.123.10
    ingressVIP: 192.168.123.11
    dnsVIP: 192.168.123.12
    bootstrapOSImage: http://ocp4-bastion.aio.example.com/images/rhcos-47.83.202105220305-0-qemu.x86_64.qcow2.gz?sha256=d3e6f4e1182789480dcb81fc8cdf37416ec9afa34b4be1056426b21b62272248
    clusterOSImage: http://ocp4-bastion.aio.example.com/images/rhcos-47.83.202105220305-0-openstack.x86_64.qcow2.gz?sha256=156e695b0a834cc9efe1ef95609cec078c65f8030be8c68b8fe946f82a65dd3a
    hosts:
      - name: master1
        role: master
        bmc:
          address: ipmi://192.168.123.1:6231
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:01
        rootDeviceHints:
          deviceName: /dev/vda
      - name: master2
        role: master
        bmc:
          address: ipmi://192.168.123.1:6232
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:02
        rootDeviceHints:
          deviceName: /dev/vda
      - name: master3
        role: master
        bmc:
          address: ipmi://192.168.123.1:6233
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:03
        rootDeviceHints:
          deviceName: /dev/vda
      - name: worker1
        role: worker
        bmc:
          address: ipmi://192.168.123.1:6234
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:04
        rootDeviceHints:
          deviceName: /dev/vda
      - name: worker2
        role: worker
        bmc:
          address: ipmi://192.168.123.1:6235
          username: admin
          password: redhat
        bootMACAddress: de:ad:be:ef:00:05
        rootDeviceHints:
          deviceName: /dev/vda
sshKey: 'REDACTED SSHKEY DATA'
pullSecret: 'REDACTED PULL-SECRET-DATA'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
REDACTED CERTIFICATE DATA
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

~~~

Now that we have examined the install-config.yaml we are ready to proceed with the deployment.  However before starting the deploy we should always make sure that the master and worker baremetal nodes we are going to use are in a powered off state:

~~~bash
[root@ocp4-bastion id]# for i in 1 2 3 4 5 6; do /usr/bin/ipmitool -I lanplus -H192.168.123.1 -p623$i -Uadmin -Predhat chassis power off; done
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
Chassis Power Control: Down/Off
[root@ocp4-bastion id]# 
~~~

## Deploying OpenShift

We're going to install the cluster using two steps, one to **create the manifests** and one to **install the cluster**. Generating the manifests separately like this isn't necessary as it is done automatically when we run a `create cluster`.  However it's interesting to be able to see these files so we've done it to allow you a chance to explore! Additionally, if you had more configuration files this would be where you would add them.

### Create the manifests

Ok, let's create our cluster state directory:

~~~bash
[root@ocp4-bastion ~]# mkdir ~/ocp-install
~~~

And place our install-config.yaml file into it:

~~~bash
[root@ocp4-bastion ~]# cp ~/lab/install-config.yaml ~/ocp-install
~~~

> **NOTE**: The installer will consume the install-config.yaml and remove the file from the state direcrtory. If you have not saved it somewhere else you can regenerate it with `openshift-baremetal-install create install-config --dir=ocp` on a running cluster.

~~~bash
[root@ocp4-bastion ~]# ~/openshift-baremetal-install --dir=ocp-install --log-level debug create manifests
DEBUG OpenShift Installer 4.7.19                   
DEBUG Built from commit 3948d1fe9f04057823b871133085d602e6575067 
DEBUG Fetching Master Machines...                  
DEBUG Loading Master Machines...                   
DEBUG   Loading Cluster ID...                      
(...)        
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
INFO Manifests created in: ocp-install/manifests and ocp-install/openshift 
~~~

If we look in the manifests directory we can see there are all sorts of configuration items.  As mentioned, additonal configruation could be placed here for customizations of the cluster before actually kicking off the deploy:

~~~bash
[root@ocp4-bastion ~]# ls -l ~/ocp-install/
total 8
drwxr-x---. 2 root root 4096 Jul 21 19:26 manifests
drwxr-x---. 2 root root 4096 Jul 21 19:26 openshift
[root@ocp4-bastion ~]# ls -l ~/ocp-install/manifests/
total 116
-rw-r-----. 1 root root  169 Jul 21 19:26 04-openshift-machine-config-operator.yaml
-rw-r-----. 1 root root 6905 Jul 21 19:26 cluster-config.yaml
-rw-r-----. 1 root root  144 Jul 21 19:26 cluster-dns-02-config.yml
-rw-r-----. 1 root root  503 Jul 21 19:26 cluster-infrastructure-02-config.yml
-rw-r-----. 1 root root  149 Jul 21 19:26 cluster-ingress-02-config.yml
-rw-r-----. 1 root root  513 Jul 21 19:26 cluster-network-01-crd.yml
-rw-r-----. 1 root root  272 Jul 21 19:26 cluster-network-02-config.yml
-rw-r-----. 1 root root  142 Jul 21 19:26 cluster-proxy-01-config.yaml
-rw-r-----. 1 root root  171 Jul 21 19:26 cluster-scheduler-02-config.yml
-rw-r-----. 1 root root  199 Jul 21 19:26 cvo-overrides.yaml
-rw-r-----. 1 root root 1335 Jul 21 19:26 etcd-ca-bundle-configmap.yaml
-rw-r-----. 1 root root 3962 Jul 21 19:26 etcd-client-secret.yaml
-rw-r-----. 1 root root 4009 Jul 21 19:26 etcd-metric-client-secret.yaml
-rw-r-----. 1 root root 1359 Jul 21 19:26 etcd-metric-serving-ca-configmap.yaml
-rw-r-----. 1 root root 3925 Jul 21 19:26 etcd-metric-signer-secret.yaml
-rw-r-----. 1 root root  156 Jul 21 19:26 etcd-namespace.yaml
-rw-r-----. 1 root root  334 Jul 21 19:26 etcd-service.yaml
-rw-r-----. 1 root root 1336 Jul 21 19:26 etcd-serving-ca-configmap.yaml
-rw-r-----. 1 root root 3898 Jul 21 19:26 etcd-signer-secret.yaml
-rw-r-----. 1 root root  289 Jul 21 19:26 image-content-source-policy-0.yaml
-rw-r-----. 1 root root  294 Jul 21 19:26 image-content-source-policy-1.yaml
-rw-r-----. 1 root root  118 Jul 21 19:26 kube-cloud-config.yaml
-rw-r-----. 1 root root 1304 Jul 21 19:26 kube-system-configmap-root-ca.yaml
-rw-r-----. 1 root root 4074 Jul 21 19:26 machine-config-server-tls-secret.yaml
-rw-r-----. 1 root root 4409 Jul 21 19:26 openshift-config-secret-pull-secret.yaml
-rw-r-----. 1 root root  201 Jul 21 19:26 openshift-kubevirt-infra-namespace.yaml
-rw-r-----. 1 root root 2480 Jul 21 19:26 user-ca-bundle-config.yaml
~~~

### Create the cluster

We have now arrived at the point where we can run the `create cluster` argument for the install command to deploy our baremetal cluster.  This process will take about ~60-90 minutes to complete so have tmux running is you want to avoid network issues causing problems! :)

> **REMINDER**: Don't forget to use tmux or screen!

~~~bash
[root@ocp4-bastion ~]# ~/openshift-baremetal-install --dir=ocp-install --log-level debug create cluster
DEBUG OpenShift Installer 4.7.19                   
DEBUG Built from commit 3948d1fe9f04057823b871133085d602e6575067 
DEBUG Fetching Metadata...                         
DEBUG Loading Metadata...                          
DEBUG   Loading Cluster ID...  
(...)
DEBUG module.masters.ironic_deployment.openshift-master-deployment[2]: Creation complete after 2m31s [id=9641e822-a747-4ae6-b139-ce1ed9869584] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[1]: Creation complete after 2m31s [id=6d4740f5-9a2a-419c-a24c-e030cf88527a] 
DEBUG module.masters.ironic_deployment.openshift-master-deployment[0]: Creation complete after 2m31s [id=f3fe15b9-d1a8-4fe2-867e-8d02489b75b6] 
DEBUG                                              
DEBUG Apply complete! Resources: 14 added, 0 changed, 0 destroyed. 
DEBUG OpenShift Installer 4.7.19                   
DEBUG Built from commit 3948d1fe9f04057823b871133085d602e6575067 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.aio.example.com:6443... 
INFO API v1.20.0+87cc9a4 up                       
INFO Waiting up to 30m0s for bootstrapping to complete... 
(...)
DEBUG Still waiting for the cluster to initialize: Working towards 4.7.19: 648 of 669 done (96% complete) 
DEBUG Still waiting for the cluster to initialize: Some cluster operators are still updating: authentication, console 
DEBUG Still waiting for the cluster to initialize: Some cluster operators are still updating: authentication, console 
DEBUG Still waiting for the cluster to initialize: Working towards 4.7.19: 651 of 669 done (97% complete) 
DEBUG Still waiting for the cluster to initialize: Some cluster operators are still updating: authentication, console 
DEBUG Still waiting for the cluster to initialize: Working towards 4.7.19: 663 of 669 done (99% complete) 
DEBUG Still waiting for the cluster to initialize: Cluster operator authentication is not available 
DEBUG Cluster is initialized                       
INFO Waiting up to 10m0s for the openshift-console route to be created... 
DEBUG Route found in openshift-console namespace: console 
DEBUG OpenShift console route is admitted          
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/ocp-install/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.aio.example.com 
INFO Login to the console with user: "kubeadmin", and password: "Qr4wP-dJy4G-QfAwP-h65hy" 
DEBUG Time elapsed per stage:                      
DEBUG     Infrastructure: 21m56s                   
DEBUG Bootstrap Complete: 9m20s                    
DEBUG  Bootstrap Destroy: 16s                      
DEBUG  Cluster Operators: 28m10s                   
INFO Time elapsed: 1h0m18s  
~~~

### Deployment Success

Once the cluster has successfully deployed at the end of the logging you will be presented with cluster command line information and also the login for the OpenShift console. Make sure to record those details somewhere convenient for later use. In the example above we seem them in these lines:

~~~bash 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.aio.example.com 
INFO Login to the console with user: "kubeadmin", and password: "Qr4wP-dJy4G-QfAwP-h65hy" 
~~~

Where the console is **https://console-openshift-console.apps.aio.example.com ** and the user to login is **kubeadmin** and the password is **Qr4wP-dJy4G-QfAwP-h65hy** (your's will be different than these of course).  The AIO environment provided the ability to deploy Guacamole which can provided UI access to the bastion host where a browser can be used to view the OpenShift console.

We can also interact with the cluster easily via the command line using the `oc` command. Before we run the `oc` commands we need to export the KUBECONFIG variable:

~~~bash
[root@ocp4-bastion ~]# export KUBECONFIG=/root/ocp-install/auth/kubeconfig
~~~

Now we can validate and confirm we have a 3 master and 2 worker cluster instantiated by issuing the `oc get nodes` command:

~~~bash
[root@ocp4-bastion ~]# oc get nodes
NAME                           STATUS   ROLES    AGE   VERSION
ocp4-master1.aio.example.com   Ready    master   50m   v1.20.0+87cc9a4
ocp4-master2.aio.example.com   Ready    master   50m   v1.20.0+87cc9a4
ocp4-master3.aio.example.com   Ready    master   50m   v1.20.0+87cc9a4
ocp4-worker1.aio.example.com   Ready    worker   31m   v1.20.0+87cc9a4
ocp4-worker2.aio.example.com   Ready    worker   31m   v1.20.0+87cc9a4
~~~

Further we can also confirm all the cluster operators are functional and available by looking at the `oc get clusteroperators` command:

~~~bash
[root@ocp4-bastion ~]# oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.7.19    True        False         False      17m
baremetal                                  4.7.19    True        False         False      49m
cloud-credential                           4.7.19    True        False         False      60m
cluster-autoscaler                         4.7.19    True        False         False      48m
config-operator                            4.7.19    True        False         False      49m
console                                    4.7.19    True        False         False      23m
csi-snapshot-controller                    4.7.19    True        False         False      39m
dns                                        4.7.19    True        False         False      48m
etcd                                       4.7.19    True        False         False      47m
image-registry                             4.7.19    True        False         False      41m
ingress                                    4.7.19    True        False         False      31m
insights                                   4.7.19    True        False         False      42m
kube-apiserver                             4.7.19    True        False         False      45m
kube-controller-manager                    4.7.19    True        False         False      46m
kube-scheduler                             4.7.19    True        False         False      46m
kube-storage-version-migrator              4.7.19    True        False         False      31m
machine-api                                4.7.19    True        False         False      45m
machine-approver                           4.7.19    True        False         False      48m
machine-config                             4.7.19    True        False         False      47m
marketplace                                4.7.19    True        False         False      47m
monitoring                                 4.7.19    True        False         False      30m
network                                    4.7.19    True        False         False      48m
node-tuning                                4.7.19    True        False         False      48m
openshift-apiserver                        4.7.19    True        False         False      41m
openshift-controller-manager               4.7.19    True        False         False      41m
openshift-samples                          4.7.19    True        False         False      33m
operator-lifecycle-manager                 4.7.19    True        False         False      48m
operator-lifecycle-manager-catalog         4.7.19    True        False         False      48m
operator-lifecycle-manager-packageserver   4.7.19    True        False         False      42m
service-ca                                 4.7.19    True        False         False      49m
storage                                    4.7.19    True        False         False      49m
~~~

Finally, we need to patch the Image Registry Operator.  In a production environment the Image Registry Operator needs to be configured with shared storage; however, since this is a lab we can just configure it with an empty directory:

~~~bash
[root@ocp4-bastion ~]# oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
 --patch '{"spec":{"storage":{"emptyDir":{}}}}'
config.imageregistry.operator.openshift.io/cluster patched
[root@ocp4-bastion ~]# oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
 --patch '{"spec":{"managementState":"Managed"}}'
config.imageregistry.operator.openshift.io/cluster patched
~~~

Once the above is configured we should be able to see the pods related to the Image Registry Operator:

~~~bash
[root@ocp4-bastion ~]# oc get pod -n openshift-image-registry
NAME                                               READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-64f5467494-tvb9s   1/1     Running   1          64m
image-registry-866ccfcdd7-nwn9m                    1/1     Running   0          20s
node-ca-57gwd                                      1/1     Running   0          33m
node-ca-9jr2l                                      1/1     Running   0          34m
node-ca-b2542                                      1/1     Running   0          43m
node-ca-dw5kq                                      1/1     Running   0          43m
node-ca-hn9vt                                      1/1     Running   0          43m
~~~

At this point you are now ready to move onto the next lab where we will look at the Machine Config Operator (aka Baremetal Operator). [Continue to the Add Worker lab!](https://github.com/RHFieldProductManagement/openshift-aio/blob/main/labs/03-addworker.md)

