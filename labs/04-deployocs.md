# Deploy OpenShift Container Storage

In the AIO deployment environment we provide 3 worker nodes to be used for consumption.  As part of that deployment we offer the capability to run OpenShift Container Storage (OCS) on top of our OpenShift deployed cluster.   AIO does have the capability to deploy OCS automatically but for those who would like to experience the process this lab gives them the hands on experience.

The mechanism for installation is to utilize the operator model and deploy from the cli. Note, it's entirely possible to deploy via the web console should you wish to do so, but we're not documenting that mechanism here.

From a node perspective, the lab environment should look like the following from a master/worker node count:

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

> **NOTE**: If you do not have **three** workers listed here, this lab will not succeed - please revert to the previous section and ensure that all three workers are provisioned, noting that the count starts from 1, hence worker1 through worker3 should be listed.

Now lets validate that each node has two extra disks which will become our OSD volumes for OCS by using the debug container on the worker node to view the block devices:

~~~bash
[root@ocp4-bastion ~]# oc debug node/ocp4-worker3.aio.example.com
Starting pod/ocp4-worker3aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.106
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-4.4# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    252:0    0   80G  0 disk 
|-vda1 252:1    0    1M  0 part 
|-vda2 252:2    0  127M  0 part 
|-vda3 252:3    0  384M  0 part /boot
|-vda4 252:4    0 79.4G  0 part /sysroot
`-vda5 252:5    0   65M  0 part 
vdb    252:16   0  100G  0 disk 
vdc    252:32   0  100G  0 disk 
sh-4.4# exit
exit
sh-4.4# exit
exit

Removing debug pod ...
~~~

We can see from the output above that on **worker3** the two **100GB** volumes exist as **vdb** and **vdc**.  Repeat the above steps to confirm that the remaining workers also have their 100GB vdb and vdc volumes attached. Once finished, we need to label our nodes for storage, this label will tell OCS that these machines can be utilised for storage requirements:

~~~bash
[root@ocp4-bastion ~]# oc label nodes ocp4-worker1.aio.example.com cluster.ocs.openshift.io/openshift-storage=''
node/ocp4-worker1.aio.example.com labeled
[root@ocp4-bastion ~]# oc label nodes ocp4-worker2.aio.example.com cluster.ocs.openshift.io/openshift-storage=''
node/ocp4-worker2.aio.example.com labeled
[root@ocp4-bastion ~]# oc label nodes ocp4-worker3.aio.example.com cluster.ocs.openshift.io/openshift-storage=''
node/ocp4-worker3.aio.example.com labeled
~~~

And confirm the label changes:

~~~bash
[root@ocp4-bastion ~]# oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
NAME                           STATUS   ROLES    AGE    VERSION
ocp4-worker1.aio.example.com   Ready    worker   5d3h   v1.20.0+87cc9a4
ocp4-worker2.aio.example.com   Ready    worker   5d3h   v1.20.0+87cc9a4
ocp4-worker3.aio.example.com   Ready    worker   20m    v1.20.0+87cc9a4
~~~



## Deploying the Local Storage Operator

Now that we know the worker nodes have their extra disk ready and labels added, we can proceed. Before installing OCS we need to first install the local storage operator which is used to consume local disks, and expose them as available persistent volumes (PV) for OCS to consume; the OCS Ceph OSD pods consume local-storage based PV's and allow additional RWX PV's to be deployed on-top.

The first step is to create an OpenShift ocal storage namespace.  We can do this by creating the following namespace yaml file:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-local-storage-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-local-storage
spec: {}
EOF
~~~

Lets take a look at the file that was created:

~~~bash
[root@ocp4-bastion ~]# cat ~/openshift-local-storage-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-local-storage
spec: {}
~~~

Now lets go ahead and create that namespace resource that the file defines:

~~~bash
[root@ocp4-bastion ~]# oc create -f ~/openshift-local-storage-namespace.yaml
namespace/openshift-local-storage created
[root@ocp4-bastion ~]# oc get namespaces openshift-local-storage
NAME                      STATUS   AGE
openshift-local-storage   Active   30s
~~~

After creating the namespace we see the details about the namespace and also can confirm that the namespace is active:

~~~bash
[root@ocp4-bastion ~]# oc describe namespaces openshift-local-storage
Name:         openshift-local-storage
Labels:       <none>
Annotations:  openshift.io/sa.scc.mcs: s0:c25,c15
              openshift.io/sa.scc.supplemental-groups: 1000630000/10000
              openshift.io/sa.scc.uid-range: 1000630000/10000
Status:       Active

No resource quota.

No LimitRange resource.
~~~

Now lets create an operator group for the OpenShift local storage operator:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-local-storage-group.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: local-operator-group
  namespace: openshift-local-storage
spec:
  targetNamespaces:
  - openshift-local-storage
EOF
~~~

Lets validate the file we just created:

~~~bash
[root@ocp4-bastion ~]# cat ~/openshift-local-storage-group.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: local-operator-group
  namespace: openshift-local-storage
spec:
  targetNamespaces:
  - openshift-local-storage
~~~

As we can see this file will create an operator group called local-operator-group under the openshift-local-storage namespace.  Lets now go ahead and create that operator group, show the operator group has been created and also show the details of the operator group:

~~~bash
[root@ocp4-bastion ~]# oc create -f ~/openshift-local-storage-group.yaml
operatorgroup.operators.coreos.com/local-operator-group created

[root@ocp4-bastion ~]# oc get operatorgroup -n openshift-local-storage
NAME                   AGE
local-operator-group   15s

[root@ocp4-bastion ~]# oc describe  operatorgroup -n openshift-local-storage
Name:         local-operator-group
Namespace:    openshift-local-storage
Labels:       <none>
Annotations:  <none>
API Version:  operators.coreos.com/v1
Kind:         OperatorGroup
Metadata:
  Creation Timestamp:  2021-07-14T20:40:29Z
  Generation:          1
  Managed Fields:
    API Version:  operators.coreos.com/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:targetNamespaces:
          .:
          v:"openshift-local-storage":
    Manager:      kubectl-create
    Operation:    Update
    Time:         2021-07-14T20:40:29Z
    API Version:  operators.coreos.com/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:lastUpdated:
        f:namespaces:
          .:
          v:"openshift-local-storage":
    Manager:         olm
    Operation:       Update
    Time:            2021-07-14T20:40:29Z
  Resource Version:  2178910
  Self Link:         /apis/operators.coreos.com/v1/namespaces/openshift-local-storage/operatorgroups/local-operator-group
  UID:               193c6aef-7fb4-4436-841b-e171cfd0eadd
Spec:
  Target Namespaces:
    openshift-local-storage
Status:
  Last Updated:  2021-07-14T20:40:29Z
  Namespaces:
    openshift-local-storage
Events:  <none>
~~~

Once the operator group has been created we need to also create a subscription for the OpenShift local storage operator:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-local-storage-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: local-storage-operator
  namespace: openshift-local-storage
spec:
  channel: "4.6"
  installPlanApproval: Automatic
  name: local-storage-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

Before we apply the subscription yaml file we created lets take a look at it:

~~~bash
[root@ocp4-bastion ~]# cat ~/openshift-local-storage-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: local-storage-operator
  namespace: openshift-local-storage
spec:
  channel: "4.6"
  installPlanApproval: Automatic
  name: local-storage-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
~~~

We can see from the above subscription yaml file we will be subscribed to the 4.6 release and the operator will run under the openshift-local-storage namespace.  Now lets go ahead and create the subscription:

~~~bash
[root@ocp4-bastion ~]# oc create -f openshift-local-storage-subscription.yaml
subscription.operators.coreos.com/local-storage-operator created
~~~

After a few minutes we should see an operator pod starting in the openshift-local-storage namespace

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-local-storage
NAME                                      READY   STATUS              RESTARTS   AGE
local-storage-operator-74c86b646c-87l2t   0/1     ContainerCreating   0          5s
~~~

We can also validate from the command line that the operator is installed and running as well:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-local-storage
NAME                                      READY   STATUS    RESTARTS   AGE
local-storage-operator-74c86b646c-87l2t   1/1     Running   0          76s
~~~

Now that we have the local storage operator installed lets make a "**LocalVolume**" storage definition file that will use the disk device in each node:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/local-storage.yaml
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/vdb
        - /dev/vdc
EOF
~~~

Lets go ahead and view the storage definitiona yaml we created:

~~~bash
[root@ocp4-bastion ~]# cat ~/local-storage.yaml
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/vdb
        - /dev/vdc
~~~

You'll see that this is set to create a local volume on every host from the block device **vdb** & **vdc** where the selector key matches **cluster.ocs.openshift.io/openshift-storage**.  *If* we had additional devices on the worker nodes for example: **vdd** and **vde**, we would just list those below the devicePaths to also be incorporated into our configuration.

Now we can go ahead and create the assets for this local-storage configuration using the local-storage.yaml we created above.

~~~bash
[root@ocp4-bastion ~]# oc create -f ~/local-storage.yaml
localvolume.local.storage.openshift.io/local-block created
~~~

If we execute an `oc get pods` command on the namespace of `local-storage` we will see containers being created in relationship to the assets from our local-storage.yaml file. The pods generated are a provisioner and diskmaker on every worker node where the node selector matched.:

~~~bash
[root@ocp4-bastion ~]# oc -n openshift-local-storage get pods
NAME                                      READY   STATUS    RESTARTS   AGE
local-block-local-diskmaker-7mjmw         1/1     Running   0          28s
local-block-local-diskmaker-bls94         1/1     Running   0          28s
local-block-local-diskmaker-qxjwh         1/1     Running   0          28s
local-block-local-provisioner-29wcm       1/1     Running   0          28s
local-block-local-provisioner-65m54       1/1     Running   0          28s
local-block-local-provisioner-95w9j       1/1     Running   0          28s
local-storage-operator-74c86b646c-87l2t   1/1     Running   0          18h
~~~

As you can see from the above we had labeled 3 worker nodes and we have 3 provisioners and 3 diskmaker pods.  To validate the nodes where the pods are running try adding `-o wide` to the command above. *Does it confirm that each worker has a provisioner and diskmaker pod?*

Furthermore we can now see that 6 PV's have been created:

~~~bash
[root@ocp4-bastion ~]# oc get pv -o wide
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE   VOLUMEMODE
local-pv-6751e4c7   100Gi      RWO            Delete           Available           localblock              34s   Block
local-pv-759d6092   100Gi      RWO            Delete           Available           localblock              44s   Block
local-pv-911d74ba   100Gi      RWO            Delete           Available           localblock              34s   Block
local-pv-9d4e1eb0   100Gi      RWO            Delete           Available           localblock              34s   Block
local-pv-be875671   100Gi      RWO            Delete           Available           localblock              44s   Block
local-pv-e2c7f8a9   100Gi      RWO            Delete           Available           localblock              34s   Block
~~~

And finally a storageclass was created for the local-storage asset we created:

~~~bash
[root@ocp4-bastion ~]# oc get sc
NAME         PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  105s
~~~


## Deploying OpenShift Container Storage

At this point in the lab we have now completed the prerequisites for OpenShift Container Storage.  The mechanism agan for installation is to utilize the operator model and deploy from the cli. Note, it's entirely possible to deploy via the web console should you wish to do so, but we're not documenting that mechanism here.

The first step in deploying OCS is to create the openshift-storage namespace.  To do that we will create a namespace yaml definition:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-storage-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    openshift.io/cluster-monitoring: "true"
  name: openshift-storage
spec: {}
EOF
~~~

Next we will create the namespace from the definition we created above:

~~~bash
[root@ocp4-bastion ~]# oc create -f ~/openshift-storage-namespace.yaml
namespace/openshift-storage created
~~~

With the namespace created we need to create and operator group an a subscription just like we did with the local storage operator.  However this time we will create both in the same file:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-storage-subscription.yaml
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
  - openshift-storage
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ocs-operator
  namespace: openshift-storage
spec:
  channel: "stable-4.6"
  installPlanApproval: Automatic
  name: ocs-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

Lets go ahead and create the operator group and subscription from the file we created:

~~~bash
[root@ocp4-bastion ~]# oc create -f ~/openshift-storage-subscription.yaml
operatorgroup.operators.coreos.com/openshift-storage-operatorgroup created
subscription.operators.coreos.com/ocs-operator created

[root@ocp4-bastion ~]# oc get pods -n openshift-storage
NAME                                   READY   STATUS              RESTARTS   AGE
noobaa-operator-5d47cf4f58-q48z5       0/1     ContainerCreating   0          19s
ocs-metrics-exporter-fbd466d84-kxn4n   0/1     ContainerCreating   0          18s
ocs-operator-7d7554cd7-5gcmr           0/1     ContainerCreating   0          19s
rook-ceph-operator-94cfd97d5-mg57r     0/1     ContainerCreating   0          19s
~~~

With the namespace, openshift-storage operator group and subscription created we should in a few minutes have a successfully running OCS operator in the environment.  We can can confirm the operator is up and running by looking at it from the command line where we should see 4 running pods under the openshift-storage namespace:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-storage
NAME                                   READY   STATUS    RESTARTS   AGE
noobaa-operator-5d47cf4f58-q48z5       1/1     Running   0          54s
ocs-metrics-exporter-fbd466d84-kxn4n   1/1     Running   0          53s
ocs-operator-7d7554cd7-5gcmr           1/1     Running   0          54s
rook-ceph-operator-94cfd97d5-mg57r     1/1     Running   0          54s
~~~

Now that we know the operator is deployed (and the associated pods are running) we can proceed to creating a hyperconverged OCS cluster.  To do this we first need to create a storage cluster yaml file that will allow us to consume each of the two pvs per worker node that we configured via the local storage operator.

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/openshift-storage-cluster.yaml
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  manageNodes: false
  resources:
    mds:
      limits:
        cpu: "3"
        memory: "8Gi"
      requests:
        cpu: "3"
        memory: "8Gi"
  monDataDirHostPath: /var/lib/rook
  managedResources:
    cephBlockPools:
      reconcileStrategy: manage
    cephFilesystems:
      reconcileStrategy: manage
    cephObjectStoreUsers:
      reconcileStrategy: manage
    cephObjectStores:
      reconcileStrategy: manage
    snapshotClasses:
      reconcileStrategy: manage
    storageClasses:
      reconcileStrategy: manage
  multiCloudGateway:
    reconcileStrategy: manage
  storageDeviceSets:
  - count: 2
    dataPVCTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: "100Gi"
        storageClassName: localblock
        volumeMode: Block
    name: ocs-deviceset
    placement: {}
    portable: false
    replica: 3
    resources:
      limits:
        cpu: "2"
        memory: "5Gi"
      requests:
        cpu: "2"
        memory: "5Gi"
EOF
~~~

If we look at the above storage cluster yaml the key things that stand out are the count, the storage size, storageclass and replica.  Because we have 2 pvs on each worker that we want included in our storage cluster from the local storage storage class we set the replica to 3 and the count to 2.   We also specify that we want to use 100Gi pvs from the localblock storage class.  We can also note the resource limits on CPU and memory to ensure that the storage cluster does not use all the resources on the worker nodes.

With the storage cluster yaml created lets go ahead and create the storage cluster:

~~~bash
[root@ocp4-bastion ~]# oc create -f openshift-storage-cluster.yaml
storagecluster.ocs.openshift.io/ocs-storagecluster created
~~~

If we run an oc get pods for the openshift-storage namespace we can see pods are starting to create to build out the storage cluster.  If one wanted to watch this continuously we could throw a watch command in front of the oc command.   It will take a few minutes to instantiate the cluster nevertheless.

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-storage
NAME                                            READY   STATUS              RESTARTS   AGE
csi-cephfsplugin-6lz7d                          0/3     ContainerCreating   0          2s
csi-cephfsplugin-pmm5g                          0/3     ContainerCreating   0          2s
csi-cephfsplugin-provisioner-749c5f9766-89vx6   0/6     ContainerCreating   0          2s
csi-cephfsplugin-provisioner-749c5f9766-ktjg8   0/6     ContainerCreating   0          1s
csi-cephfsplugin-ptptq                          0/3     ContainerCreating   0          2s
csi-rbdplugin-5wtcw                             0/3     ContainerCreating   0          3s
csi-rbdplugin-provisioner-79c88978ff-5b9nm      0/6     ContainerCreating   0          2s
csi-rbdplugin-provisioner-79c88978ff-s6zwj      0/6     ContainerCreating   0          2s
csi-rbdplugin-qgthm                             0/3     ContainerCreating   0          3s
csi-rbdplugin-xrjvq                             0/3     ContainerCreating   0          3s
noobaa-operator-5d47cf4f58-q48z5                1/1     Running             0          13m
ocs-metrics-exporter-fbd466d84-kxn4n            1/1     Running             0          13m
ocs-operator-7d7554cd7-5gcmr                    1/1     Running             0          13m
rook-ceph-detect-version-cb9v9                  0/1     PodInitializing     0          6s
rook-ceph-operator-94cfd97d5-mg57r              1/1     Running             0          13m
~~~

Finally after about 5 minutes we can see all the pods that have been generated to deploy the OCS storage cluster:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-storage
NAME                                                              READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-6lz7d                                            3/3     Running     0          6m13s
csi-cephfsplugin-pmm5g                                            3/3     Running     0          6m13s
csi-cephfsplugin-provisioner-749c5f9766-89vx6                     6/6     Running     0          6m13s
csi-cephfsplugin-provisioner-749c5f9766-ktjg8                     6/6     Running     0          6m12s
csi-cephfsplugin-ptptq                                            3/3     Running     0          6m13s
csi-rbdplugin-5wtcw                                               3/3     Running     0          6m14s
csi-rbdplugin-provisioner-79c88978ff-5b9nm                        6/6     Running     0          6m13s
csi-rbdplugin-provisioner-79c88978ff-s6zwj                        6/6     Running     0          6m13s
csi-rbdplugin-qgthm                                               3/3     Running     0          6m14s
csi-rbdplugin-xrjvq                                               3/3     Running     0          6m14s
noobaa-core-0                                                     1/1     Running     0          3m28s
noobaa-db-0                                                       1/1     Running     0          3m28s
noobaa-endpoint-8448cb4bcb-qgddv                                  1/1     Running     0          39s
noobaa-operator-5d47cf4f58-q48z5                                  1/1     Running     0          19m
ocs-metrics-exporter-fbd466d84-kxn4n                              1/1     Running     0          19m
ocs-operator-7d7554cd7-5gcmr                                      1/1     Running     0          19m
rook-ceph-crashcollector-ocp4-worker1.aio.example.com-7556jrn5l   1/1     Running     0          4m14s
rook-ceph-crashcollector-ocp4-worker2.aio.example.com-5d65n7v7m   1/1     Running     0          5m9s
rook-ceph-crashcollector-ocp4-worker3.aio.example.com-65d4zvppx   1/1     Running     0          5m27s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-7b87d55f2kfff   1/1     Running     0          3m9s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-5cddbbddp2mkl   1/1     Running     0          3m6s
rook-ceph-mgr-a-7f796dc664-lxqqg                                  1/1     Running     0          3m51s
rook-ceph-mon-a-74f59544d8-kkb6m                                  1/1     Running     0          5m27s
rook-ceph-mon-b-7b5cff9b5f-c5lck                                  1/1     Running     0          5m9s
rook-ceph-mon-c-5476944976-ggvlq                                  1/1     Running     0          4m14s
rook-ceph-operator-94cfd97d5-mg57r                                1/1     Running     0          19m
rook-ceph-osd-0-6b69bb668d-gkjw9                                  1/1     Running     0          3m35s
rook-ceph-osd-1-5c98954c75-wskr4                                  1/1     Running     0          3m34s
rook-ceph-osd-2-c998f4d95-5kxdr                                   1/1     Running     0          3m34s
rook-ceph-osd-3-5575c6d996-cktmk                                  1/1     Running     0          3m33s
rook-ceph-osd-4-7c7cfb6df-g28mx                                   1/1     Running     0          3m31s
rook-ceph-osd-5-84b5fffc6c-vczjc                                  1/1     Running     0          3m32s
rook-ceph-osd-prepare-ocs-deviceset-0-data-0g7jnx-h7pws           0/1     Completed   0          3m49s
rook-ceph-osd-prepare-ocs-deviceset-0-data-1rvzs6-954f2           0/1     Completed   0          3m49s
rook-ceph-osd-prepare-ocs-deviceset-1-data-0ffs9p-zqqpp           0/1     Completed   0          3m48s
rook-ceph-osd-prepare-ocs-deviceset-1-data-1sxx7p-gk65z           0/1     Completed   0          3m48s
rook-ceph-osd-prepare-ocs-deviceset-2-data-0pp4gm-j27kq           0/1     Completed   0          3m48s
rook-ceph-osd-prepare-ocs-deviceset-2-data-18bcdc-btf54           0/1     Completed   0          3m47s
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-a-5d89cb99lbb9   1/1     Running     0          2m33s
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-b-5bb55f6rhlmx   1/1     Running     0          2m27s
~~~

We can also confirm this from the command line by issuing an oc get storageclass.  We should see 4 new storageclasses: one for block (rbd), two for object (rgw/nooba) and one for file(cephfs).  

~~~bash
[root@ocp4-bastion ~]# oc get storageclass
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock                    kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  3h4m
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   6m50s
ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  6m50s
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   6m50s
openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate              false                  70s
~~~

At this point you have a fully functional OpenShift Container Storage cluster to be consumed by applications. You can optionally deploy the Ceph tools pod, where you can dive into some of the Ceph internals at your leisure:

~~~bash
[root@ocp4-bastion ~]# oc patch OCSInitialization ocsinit -n openshift-storage \
>     --type json --patch '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
ocsinitialization.ocs.openshift.io/ocsinit patched
~~~

If you look at the pods, you'll find a newly started Ceph tools pod:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-storage | grep rook-ceph-tools
rook-ceph-tools-8567c77f6f-5kmch                                  1/1     Running     0          21s
~~~

If you '*exec*' into this pod, you'll find a configured environment ready to issue Ceph commands.  Further executing a ceph -s (status) will show that there are 6 OSDs in the storage cluster with roughly 600Gi of storage available:

~~~bash
[root@ocp4-bastion ~]# oc exec -it rook-ceph-tools-8567c77f6f-5kmch -n openshift-storage bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
[root@ocp4-worker2 /]# ceph -s
  cluster:
    id:     d8003133-d30f-4029-ab9a-e15219faf50b
    health: HEALTH_WARN
            clock skew detected on mon.b, mon.c
 
  services:
    mon: 3 daemons, quorum a,b,c (age 6m)
    mgr: a(active, since 5m)
    mds: ocs-storagecluster-cephfilesystem:1 {0=ocs-storagecluster-cephfilesystem-a=up:active} 1 up:standby-replay
    osd: 6 osds: 6 up (since 5m), 6 in (since 5m)
    rgw: 2 daemons active (ocs.storagecluster.cephobjectstore.a, ocs.storagecluster.cephobjectstore.b)
 
  data:
    pools:   10 pools, 368 pgs
    objects: 321 objects, 85 MiB
    usage:   6.1 GiB used, 594 GiB / 600 GiB avail
    pgs:     368 active+clean
 
  io:
    client:   937 B/s rd, 11 KiB/s wr, 1 op/s rd, 1 op/s wr
 
[root@ocp4-worker2 /]# exit
exit
[root@ocp4-bastion ~]# 
~~~

> **NOTE**: Make sure you exit out of this pod before continuing.



Now it's time to do some fun stuff with all this infrastructure! [In the next lab you will get to deploy OpenShift Virtualization!](https://github.com/RHFieldProductManagement/openshift-aio/blob/main/labs/05-deploycnv.md)
