# Deploy OpenShift Container Storage

Now that we have added an additional worker node to our lab cluster environment we can deploy OpenShift Container Storage (OCS) on top.  The mechanism for installation is to utilise the operator model and deploy via the OpenShift Operator Hub (Marketplace) in the web-console. Note, it's entirely possible to deploy via the CLI should you wish to do so, but we're not documenting that mechanism here. However, we will leverage command line and web-console to show the progress of the deployment.

From a node perspective, the lab environment should look like the following from a master/worker node count:

~~~bash
[lab-user@provision ~]$ oc get nodes
NAME                                 STATUS   ROLES    AGE   VERSION
master-0.dtchw.dynamic.opentlc.com   Ready    master   63m   v1.18.3+47c0e71
master-1.dtchw.dynamic.opentlc.com   Ready    master   63m   v1.18.3+47c0e71
master-2.dtchw.dynamic.opentlc.com   Ready    master   63m   v1.18.3+47c0e71
worker-0.dtchw.dynamic.opentlc.com   Ready    worker   44m   v1.18.3+47c0e71
worker-1.dtchw.dynamic.opentlc.com   Ready    worker   43m   v1.18.3+47c0e71
worker-2.dtchw.dynamic.opentlc.com   Ready    worker   23m   v1.18.3+47c0e71
~~~

> **NOTE**: If you do not have **three** workers listed here, this lab will not succeed - please revert to the previous section and ensure that all three workers are provisioned, noting that the count starts from 0, hence worker-0 through 2 should be listed.

We need to attach a 100GB disk to each of our worker nodes in the lab environment; these disks will provide the storage capacity for the OCS based disks (or Ceph OSD's under the covers).  Thankfully we have a little script to do this for us on the provisioning node; utilising the OpenStack API to attach Cinder volumes to our virtual worker nodes, mimicking a real baremetal node with spare, unused disks. Before we run the script lets take a look at it, you'll note that the script has already been automatically customised to suit your environment:

~~~bash
[lab-user@provision ~]$ cat ~/scripts/10_volume-attach.sh
#!/bin/bash
OSP_PROJECT="dtchw-project"
GUID="dtchw"

attach() {
  for NODE in $( openstack --os-cloud=$OSP_PROJECT server list|grep worker|cut -d\| -f3|sed 's/ //g' )
  do
          openstack --os-cloud=$OSP_PROJECT server add volume $NODE $NODE-volume
  done
}

detach() {
  for NODE in $( openstack --os-cloud=$OSP_PROJECT server list|grep worker|cut -d\| -f3|sed 's/ //g' )
  do
          openstack --os-cloud=$OSP_PROJECT server remove volume $NODE $NODE-volume
  done
}

poweroff() {
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6200 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6201 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6202 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6203 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6204 -Uadmin -Predhat chassis power off
  /usr/bin/ipmitool -I lanplus -H10.20.0.3 -p6205 -Uadmin -Predhat chassis power off
}

case $1 in
  attach) attach ;;
  detach) poweroff
          sleep 10
          detach ;;
  power) poweroff ;;
  *) attach ;;
esac
~~~

We can see the script will attach and detach volumes from a list of worker nodes within a given lab environment, and can also be used to detach if required, which will power the nodes down first, but for now all we want to do is attach them, which is the default behaviour of the script. Lets go ahead and run the script:

~~~bash
[lab-user@provision ~]$ unset OS_URL OS_TOKEN
[lab-user@provision ~]$ ~/scripts/10_volume-attach.sh
(no output)
[lab-user@provision ~]$ echo $?
0
~~~

> **NOTE**: Ensure that you have a '0' return code and not something else - zero means that the attachment was succes

We can validate that each node has the extra disk by using the debug container on the worker node:

~~~bash
[lab-user@provision ~]$ oc debug node/worker-0.$GUID.dynamic.opentlc.com
Starting pod/worker-0-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.20.0.200
If you don't see a command prompt, try pressing enter.
sh-4.2# 
~~~

Once inside the debug container we can look at the block devices available:

~~~bash
sh-4.2# chroot /host
sh-4.4# lsblk
NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda                          252:0    0  100G  0 disk 
|-sda1                       252:1    0  384M  0 part /boot
|-sda2                       252:2    0  127M  0 part /boot/efi
|-sda3                       252:3    0    1M  0 part 
|-sda4                       252:4    0 99.4G  0 part 
| `-coreos-luks-root-nocrypt 253:0    0 99.4G  0 dm   /sysroot
`-sda5                       252:5    0   65M  0 part 
sdb                          252:16   0  100G  0 disk 
sh-4.4# exit
exit
sh-4.2# exit
exit

Removing debug pod ...
~~~

We can see from the output above that on **worker-0** the new **100GB** volume was attached as **sdb**.  Repeat the above steps to confirm that the remaining workers also have their 100GB sdb volume attached. Once finished, we need to label our nodes for storage, this label will tell OCS that these machines can be utilised for storage requirements:

~~~bash
[lab-user@provision ~]$ oc label nodes worker-0.$GUID.dynamic.opentlc.com cluster.ocs.openshift.io/openshift-storage=''
node/worker-0 labeled

[lab-user@provision ~]$ oc label nodes worker-1.$GUID.dynamic.opentlc.com cluster.ocs.openshift.io/openshift-storage=''
node/worker-1 labeled

[lab-user@provision ~]$ oc label nodes worker-2.$GUID.dynamic.opentlc.com cluster.ocs.openshift.io/openshift-storage=''
node/worker-2 labeled
~~~

And confirm the change:

~~~bash
[lab-user@provision ~]$ oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
NAME                                 STATUS   ROLES    AGE   VERSION
worker-0.dtchw.dynamic.opentlc.com   Ready    worker   47m   v1.18.3+47c0e71
worker-1.dtchw.dynamic.opentlc.com   Ready    worker   46m   v1.18.3+47c0e71
worker-2.dtchw.dynamic.opentlc.com   Ready    worker   26m   v1.18.3+47c0e71
~~~



## Deploying the Local Storage Operator

Now that we know the worker nodes have their extra disk ready and labels added, we can proceed. Before installing OCS we need to first install the **[local-storage operator](https://github.com/openshift/local-storage-operator)** which is used to consume local disks, and expose them as available persistent volumes (PV) for OCS to consume; the OCS Ceph OSD pods consume local-storage based PV's and allow additional RWX PV's to be deployed on-top.

The first step is to create a local storage namespace in the OpenShift console.  Navigate to **Administration** -> **Namespaces** and click on the create namespace button.  Once the below dialogue appears, set the namespace Name to **local-storage**.

<img src="img/create-local-storage-namespace.png"/>

After creating the namespace we see the details about the namespace and also can confirm that the namespace is active:

<img src="img/show-local-storage-namespace.png"/>

Now we can go to **Operators** -> **OperatorHub** and search for the local storage operator in the catalog:

<img src="img/select-local-storage-operator.png"/>

If you do not see any available operators in the operator hub, the marketplace pods may need restarting. This is sometimes related to the initial timeout of the installation (if yours timed out):

~~~bash
[lab-user@provision ~]$ for i in $(oc get pods -A | awk '/marketplace/ {print $2;}');
   do oc delete pod $i -n openshift-marketplace; done

pod "certified-operators-7fd49b9b57-jgbkq" deleted
pod "community-operators-748759c64d-c8cmb" deleted
pod "marketplace-operator-d7494b45f-qdgd5" deleted
pod "redhat-marketplace-b96c44cdd-fhvhq" deleted
pod "redhat-operators-585c89dcf9-6wcrr" deleted
~~~

You'll need to wait a minute or two, and then try refreshing the operator hub page. You should then be able to search for "local storage" and proceed.

Select the local storage operator and click install:

<img src="img/install-local-storage-operator.png"/>

This will bring up a dialogue of options for configuring the operator before deploying.  The defaults are usually accceptable but note that you can configure the version, installation mode, namespace where operator should run and the approval strategy.

Select the defaults (ensuring that the **local-storage** namespace is selected) and click install:

<img src="img/install-choices-local-storage-operator.png"/>

Once the operator is installed we can navigate to **Operators** --> **Installed Operators** and see the local storage operator is installing which will eventually turn to a suceeded when complete:

<img src="img/installing-status-local-storage-operator.png"/>

We can also validate from the command line that the operator is installed and running as well:

~~~bash
[lab-user@provision ~]$ oc get pods -n local-storage
local-storage-operator-57455d9cb4-4tj54   1/1     Running   0          10m
~~~

Now that we have the local storage operator installed lets make a "**LocalVolume**" storage definition file that will use the disk device in each node:

~~~bash
[lab-user@provision scripts]$ cat << EOF > ~/local-storage.yaml
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: local-storage
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
        - /dev/sdb
EOF
~~~

You'll see that this is set to create a local volume on every host from the block device **sdb** where the selector key matches **cluster.ocs.openshift.io/openshift-storage**.  *If* we had additional devices on the worker nodes for example: **sdc** and sdd, we would just list those below the devicePaths to also be incorporated into our configuration.

At this point we should double-check that all three of our worker nodes have the the OCS storage label.

~~~bash
[lab-user@provision scripts]$ oc get nodes -l cluster.ocs.openshift.io/openshift-storage
NAME                                 STATUS   ROLES    AGE    VERSION
worker-0.prt8x.dynamic.opentlc.com   Ready    worker   136m   v1.18.3+47c0e71
worker-1.prt8x.dynamic.opentlc.com   Ready    worker   136m   v1.18.3+47c0e71
worker-2.prt8x.dynamic.opentlc.com   Ready    worker   75m    v1.18.3+47c0e71
~~~

Now we can go ahead and create the assets for this local-storage configuration using the local-storage.yaml we created above.

~~~bash
[lab-user@provision ~]$ oc create -f ~/local-storage.yaml
localvolume.local.storage.openshift.io/local-block created
~~~

If we execute an `oc get pods` command on the namespace of `local-storage` we will see containers being created in relationship to the assets from our local-storage.yaml file. The pods generated are a provisioner and diskmaker on every worker node where the node selector matched.:

~~~bash
[lab-user@provision ~]$ oc -n local-storage get pods
NAME                                      READY   STATUS              RESTARTS   AGE
local-block-local-diskmaker-626kf         0/1     ContainerCreating   0          8s
local-block-local-diskmaker-w5l5h         0/1     ContainerCreating   0          9s
local-block-local-diskmaker-xrxmh         0/1     ContainerCreating   0          9s
local-block-local-provisioner-9mhdq       0/1     ContainerCreating   0          9s
local-block-local-provisioner-lw9fm       0/1     ContainerCreating   0          9s
local-block-local-provisioner-xhf2x       0/1     ContainerCreating   0          9s
local-storage-operator-57455d9cb4-4tj54   1/1     Running             0          76m
~~~

As you can see from the above we had labeled 3 worker nodes and we have 3 provisioners and 3 diskmaker pods.  To validate the nodes where the pods are running try adding `-o wide` to the command above. *Does it confirm that each worker has a provisioner and diskmaker pod?*

Furthermore we can now see that 3 PV's have been created, making up our **100GB sdb** disks we attached at the beginning of the lab:

~~~bash
[lab-user@provision ~]$ oc get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-40d06fba   100Gi      RWO            Delete           Available           localblock              22s
local-pv-8aea98b7   100Gi      RWO            Delete           Available           localblock              22s
local-pv-e62c1b44   100Gi      RWO            Delete           Available           localblock              22s
~~~

And finally a storageclass was created for the local-storage asset we created:

~~~bash
[lab-user@provision scripts]$ oc get sc
NAME         PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  4m37s
~~~



## Deploying OpenShift Container Storage

At this point in the lab we have now completed the prerequisites for OpenShift Container Storage.  We can now move onto actually installing the OCS operator.  To do that we will use Operator Hub inside of the OpenShift Web Console once again.  Navigate to **Operators** --> OperatorHub and then search for **OpenShift Container Storage**:

<img src="img/ocs-operator.png"/>

Click on the operator and then select install:

<img src="img/install-ocs-operator.png"/>

The operator installation will present you with options similiar to what we saw when we installed the local storage operator.  Again we have the ability here to select version, installation mode, namespace and approval strategy.  As with the previous operator we will use the defaults as they are presented and click install:

<img src="img/options-install-ocs-operator.png"/>

Once the operator has been installed the OpenShift Console will display the OCS operator and the status of successfully installed:

<img src="img/success-install-ocs-operator.png"/>

We can further confirm the operator is up and running by looking at it from the command line where we should see 3 running pods under the openshift-storage namespace:

~~~bash
[lab-user@provision ~]$ oc get pods -n openshift-storage
NAME                                  READY   STATUS    RESTARTS   AGE
noobaa-operator-5567695698-fc8t6      1/1     Running   0          10m
ocs-operator-6888cb5bdf-7w6ct         1/1     Running   0          10m
rook-ceph-operator-7bdb4cd5d9-qmggh   1/1     Running   5          10m
~~~

Now that we know the operator is deployed (and the associated pods are running) we can go back to the OpenShift Console and click on the operator to bring us into an operator details page.  Here we will want to click on Create Instance in the "**Storage Cluster**" box:

<img src="img/details-ocs-operator.png"/>

This will bring up the options page for creating a storage cluster that can either be external or internal; starting with OCS 4.5 we introduced the ability to consume an externally provisioned cluster, but in our case we are going to use internal because we have the necessary resources in our environment to create a cluster.

Further down the page you will notice 3 worker nodes are checked as these were the nodes we labeled earlier for OCS.  In the **Storage Class** drop down we have only one option to chose and that is **localblock** which was the storage class we created with the local-storage operator and assets created above. Once you select that as the storage class the capacity and replicas are shown below the storageclass box. Notice the capacity is **300GB**, the sum of our 3x100GB volumes across our three nodes. Once everything is selected we can click on create:

<img src="img/options-create-cluster-ocs-operator.png"/>

If we quickly jump over to the command line and issue a oc get pods on the openshift-storage namespace we can see new containers are being created, you may need to run this a few times to see some of the pods starting up:

~~~bash
[lab-user@provision ~]$ oc get pods -n openshift-storage
NAME                                            READY   STATUS              RESTARTS   AGE
csi-cephfsplugin-6mk78                          0/3     ContainerCreating   0          21s
csi-cephfsplugin-9fglq                          0/3     ContainerCreating   0          21s
csi-cephfsplugin-lsldk                          0/3     ContainerCreating   0          20s
csi-cephfsplugin-provisioner-5f8b66cc96-2z755   0/5     ContainerCreating   0          20s
csi-cephfsplugin-provisioner-5f8b66cc96-wsnsb   0/5     ContainerCreating   0          19s
csi-rbdplugin-b4qh5                             0/3     ContainerCreating   0          22s
csi-rbdplugin-k2fjg                             3/3     Running             0          23s
csi-rbdplugin-lnpgn                             0/3     ContainerCreating   0          22s
csi-rbdplugin-provisioner-66f66699c8-l9g9l      0/5     ContainerCreating   0          22s
csi-rbdplugin-provisioner-66f66699c8-v7ghq      0/5     ContainerCreating   0          21s
noobaa-operator-5567695698-fc8t6                1/1     Running             0          104m
ocs-operator-6888cb5bdf-7w6ct                   0/1     Running             0          104m
rook-ceph-mon-a-canary-587d74787d-wt247         0/1     ContainerCreating   0          8s
rook-ceph-mon-b-canary-6fd99d6865-fgcpx         0/1     ContainerCreating   0          3s
rook-ceph-operator-7bdb4cd5d9-qmggh             1/1     Running             5          104m
~~~

Finally we should see in the OpenShift Console that the cluster is marked as "**ready**", noting that it may say "**progressing**" for a good few minutes.  This tells us that all the pods required to form the cluster are running and active.

<img src="img/success-cluster-create-ocs-operator.png"/>

We can further confirm this from the command line by issuing the same `oc get pods` command against the openshift-storage namespace. Can you see there is a **mon** (monitor) and **osd** (object storage daemon) pod on each of our three worker nodes?

~~~bash
[lab-user@provision ~]$ oc get pods -n openshift-storage
NAME                                                              READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-6mk78                                            3/3     Running     0          5m1s
csi-cephfsplugin-9fglq                                            3/3     Running     0          5m1s
csi-cephfsplugin-lsldk                                            3/3     Running     0          5m
csi-cephfsplugin-provisioner-5f8b66cc96-2z755                     5/5     Running     0          5m
csi-cephfsplugin-provisioner-5f8b66cc96-wsnsb                     5/5     Running     0          4m59s
csi-rbdplugin-b4qh5                                               3/3     Running     0          5m2s
csi-rbdplugin-k2fjg                                               3/3     Running     0          5m3s
csi-rbdplugin-lnpgn                                               3/3     Running     0          5m2s
csi-rbdplugin-provisioner-66f66699c8-l9g9l                        5/5     Running     0          5m2s
csi-rbdplugin-provisioner-66f66699c8-v7ghq                        5/5     Running     0          5m1s
noobaa-core-0                                                     1/1     Running     0          2m38s
noobaa-db-0                                                       1/1     Running     0          2m38s
noobaa-endpoint-74b9ddcffc-pcg95                                  1/1     Running     0          41s
noobaa-operator-5567695698-fc8t6                                  1/1     Running     0          109m
ocs-operator-6888cb5bdf-7w6ct                                     1/1     Running     0          109m
rook-ceph-crashcollector-worker-0-6f9d88bcb6-79bmf                1/1     Running     0          3m23s
rook-ceph-crashcollector-worker-1-78b56fbfc5-lqsnl                1/1     Running     0          3m45s
rook-ceph-crashcollector-worker-2-77687b587b-j4h2h                1/1     Running     0          3m53s
rook-ceph-drain-canary-worker-0-5d9d5b4977-8tc62                  1/1     Running     0          2m43s
rook-ceph-drain-canary-worker-1-54644f5c94-mhnzt                  1/1     Running     0          2m44s
rook-ceph-drain-canary-worker-2-598f89d79f-gbcjc                  1/1     Running     0          2m41s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-dd756495qt9n4   1/1     Running     0          2m6s
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-584dd7bc9hsr4   1/1     Running     0          2m5s
rook-ceph-mgr-a-848f8c59fd-ln6nz                                  1/1     Running     0          3m3s
rook-ceph-mon-a-5896d85b4f-tgc64                                  1/1     Running     0          3m54s
rook-ceph-mon-b-7695696f7d-9l565                                  1/1     Running     0          3m45s
rook-ceph-mon-c-5d95b547dc-gsjhj                                  1/1     Running     0          3m24s
rook-ceph-operator-7bdb4cd5d9-qmggh                               1/1     Running     5          109m
rook-ceph-osd-0-745d68b9df-qglsp                                  1/1     Running     0          2m45s
rook-ceph-osd-1-7946dc4cbc-sxm5w                                  1/1     Running     0          2m43s
rook-ceph-osd-2-85746d789-hhj5n                                   1/1     Running     0          2m42s
rook-ceph-osd-prepare-ocs-deviceset-0-data-0-vw4rb-q9kqg          0/1     Completed   0          2m59s
rook-ceph-osd-prepare-ocs-deviceset-1-data-0-znnlk-cghzf          0/1     Completed   0          2m58s
rook-ceph-osd-prepare-ocs-deviceset-2-data-0-7wdbq-r26b8          0/1     Completed   0          2m58s
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-a-549f6f742cgb   1/1     Running     0          87s
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-b-9b695d7q8tml   1/1     Running     0          83s
~~~

In the OpenShift Web Console if we go to **Storage** --> **Storage Classes** we can also see additional storage classes related to OCS have been created:

<img src="img/storageclasses-ocs-operator.png"/>

We can also confirm this from the command line by issuing an oc get storageclass.  We should see 4 new storageclasses: one for block (rbd), two for object (rgw/nooba) and one for file(cephfs).  

~~~bash
[lab-user@provision ~]$ oc get storageclass
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock                    kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  115m
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   5m36s
ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  5m36s
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   5m36s
openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate              false                  65s
~~~

At this point you have a fully functional OpenShift Container Storage cluster to be consumed by applications. You can optionally deploy the Ceph tools pod, where you can dive into some of the Ceph internals at your leisure:

~~~bash
[lab-user@provision ~]$ oc patch OCSInitialization ocsinit -n openshift-storage \
    --type json --patch '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'

ocsinitialization.ocs.openshift.io/ocsinit patched
~~~

If you look at the pods, you'll find a newly started Ceph tools pod:

~~~bash
[lab-user@provision ~]$ oc get pods -n openshift-storage | grep rook-ceph-tools
rook-ceph-tools-7fcff79f44-g9r6t                                  1/1     Running     0          73s
~~~

If you '*exec*' into this pod, you'll find a configured environment ready to issue Ceph commands:

~~~bash
[lab-user@provision ~]$ oc exec -it rook-ceph-tools-7fcff79f44-g9r6t -n openshift-storage bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
[root@worker-0 /]# ceph -s
  cluster:
    id:     29fe66fc-b098-4f7d-9e69-b3a9ce661444
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 5m)
    mgr: a(active, since 5m)
    mds: ocs-storagecluster-cephfilesystem:1 {0=ocs-storagecluster-cephfilesystem-a=up:active} 1 up:standby-replay
    osd: 3 osds: 3 up (since 5m), 3 in (since 5m)
    rgw: 2 daemons active (ocs.storagecluster.cephobjectstore.a, ocs.storagecluster.cephobjectstore.b)

  task status:
    scrub status:
        mds.ocs-storagecluster-cephfilesystem-a: idle
        mds.ocs-storagecluster-cephfilesystem-b: idle

  data:
    pools:   10 pools, 176 pgs
    objects: 301 objects, 83 MiB
    usage:   3.1 GiB used, 297 GiB / 300 GiB avail
    pgs:     176 active+clean

  io:
    client:   1.3 KiB/s rd, 7.5 KiB/s wr, 2 op/s rd, 0 op/s wr

[root@worker-0 /]# exit
exit
[lab-user@provision ~]$
~~~

> **NOTE**: Make sure you exit out of this pod before continuing.



Now it's time to do some fun stuff with all this infrastructure! [In the next lab you will get to deploy OpenShift Virtualization!](https://github.com/RHFieldProductManagement/baremetal-ipi-lab/blob/master/08-deploycnv.md)
