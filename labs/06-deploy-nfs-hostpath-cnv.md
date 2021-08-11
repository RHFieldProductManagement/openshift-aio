# **Deploying NFS Shared Storage and HostPath Storage**

In this lab we are going to setup two different types of storage in addition to OCS/ODF which may have been configured either via AIO or a previous lab. The first storage is standard NFS shared storage, and the second is `hostpath` storage which uses the hypervisor's local disks, somewhat akin to ephemeral storage provided by OpenStack.

First for NFS storage ensure we are in the default project:

~~~bash
[root@ocp4-bastion ~]# oc project default
Already on project "default" on server "https://api.aio.example.com:6443".
~~~

>**NOTE**: If you don't use the default project for the next few lab steps, it's likely that you'll run into some errors - some resources are scoped, i.e. aligned to a namespace, and others are not. Ensuring you're in the default namespace now will ensure that all of the coming lab steps should flow together.

Now let's setup a storage class for NFS backed volumes, utilising the `kubernetes.io/no-provisioner` as the provisioner; to be clear this mean **no** dynamic provisioning of volumes on demand of a PVC:

~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: nfs
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Delete
EOF

storageclass.storage.k8s.io/nfs created

[root@ocp4-bastion ~]# oc get sc nfs
NAME   PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs    kubernetes.io/no-provisioner   Delete          Immediate           false                  20s
~~~


Next we will create two physical volumes (PVs): **nfs-pv1** and **nfs-pv2**:

Create nfs-pv1 with the following resource yaml:

~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 40Gi
  nfs:
    path: /nfs/pv1
    server: 192.168.123.100
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
EOF

persistentvolume/nfs-pv1 created
~~~

Create nfs-pv2 with the following resource yaml:

~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv2
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 40Gi
  nfs:
    path: /nfs/pv2
    server: 192.168.123.100
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
EOF

persistentvolume/nfs-pv2 created
~~~

Check that the PV's are created and available:

~~~bash
[root@ocp4-bastion ~]# oc get pv |grep nfs
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM       STORAGECLASS                  REASON   AGE
nfs-pv1    40Gi       RWO,RWX        Delete           Available               nfs                                    3m32s
nfs-pv2    40Gi       RWO,RWX        Delete           Available               nfs                                    3m3s
~~~

Now let's create a new NFS-based Peristent Volume Claim (PVC). 

For this volume claim we will use a special annotation `cdi.kubevirt.io/storage.import.endpoint` which utilises the Kubernetes Containerized Data Importer (CDI). 

> **NOTE**: CDI is a utility to import, upload, and clone virtual machine images for OpenShift virtualisation. The CDI controller watches for this annotation on the PVC and if found it starts a process to import, upload, or clone. When the annotation is detected the `CDI` controller starts a pod which imports the image from that URL. Cloning and uploading follow a similar process. Read more about the Containerised Data Importer [here](https://github.com/kubevirt/containerized-data-importer).

Basically we are asking OpenShift to create this PVC and use the image in the endpoint to fill it. In this case we use `"http://192.168.123.100:81/rhel8-kvm.img"` in the annotation to ensure that upon instantiation of the PV it is populated with the contents of our specific RHEL8 KVM image.

In addition to triggering the CDI utility we also specify the storage class we created earlier (`nfs`) which is setting the `kubernetes.io/no-provisioner` type as described. Finally note the `requests` section. We are asking for a 40gb volume size which we ensured were available previously via nfs-pv1 and nfs-pv2.

OK, let's create the PVC with all this included!

~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "rhel8-nfs"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "http://192.168.123.100:81/rhel8-kvm.img"
spec:
  volumeMode: Filesystem
  storageClassName: nfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 40Gi
EOF
persistentvolumeclaim/rhel8-nfs created
~~~

Once created, CDI triggers the importer pod automatically to take care of the conversion for you:

~~~bash
[root@ocp4-bastion ~]# oc get pods
NAME                 READY   STATUS              RESTARTS   AGE
importer-rhel8-nfs   0/1     ContainerCreating   0          15s
~~~

Watch the logs and you can see the process:

~~~bash
[root@ocp4-bastion ~]# oc logs importer-rhel8-nfs -f
I0722 21:08:31.696805       1 importer.go:52] Starting importer
I0722 21:08:31.698159       1 importer.go:134] begin import process
I0722 21:08:31.702940       1 data-processor.go:324] Calculating available size
I0722 21:08:31.703825       1 data-processor.go:336] Checking out file system volume size.
I0722 21:08:31.704194       1 data-processor.go:344] Request image size not empty.
I0722 21:08:31.704222       1 data-processor.go:349] Target size 40Gi.
I0722 21:08:31.705436       1 data-processor.go:232] New phase: TransferDataFile
I0722 21:08:31.727374       1 util.go:161] Writing data...
I0722 21:08:32.705842       1 prometheus.go:69] 3.61
I0722 21:08:33.706218       1 prometheus.go:69] 6.86
I0722 21:08:34.706365       1 prometheus.go:69] 9.20
(...)
I0722 21:09:21.732036       1 prometheus.go:69] 95.42
I0722 21:09:22.732273       1 prometheus.go:69] 98.01
I0722 21:09:23.732657       1 prometheus.go:69] 100.00
I0722 21:09:37.952573       1 data-processor.go:232] New phase: Resize
W0722 21:09:39.094796       1 data-processor.go:311] Available space less than requested size, resizing image to available space 40587440640.
I0722 21:09:39.094825       1 data-processor.go:317] Expanding image size to: 40587440640
I0722 21:09:39.120150       1 data-processor.go:238] Validating image
I0722 21:09:39.134054       1 data-processor.go:232] New phase: Complete
I0722 21:09:39.134248       1 importer.go:212] Import Complete
~~~

Once this process has completed the PVC is ready to be consumed:

~~~bash
[root@ocp4-bastion ~]# oc get pvc
NAME        STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rhel8-nfs   Bound    nfs-pv1   40Gi       RWO,RWX        nfs            5m11s
~~~

> **NOTE**: In your environment it may be bound to `nfs-pv1` or `nfs-pv2` - we simply created two earlier for convenience, so don't worry if your output is not exactly the same here. The important part is that it's showing as `Bound`.

This same configuration should be reflected when asking OpenShift for a list of persistent volumes (`PV`), noting that one of the PV's we created earlier is still showing as `Available` and one is `Bound` to the claim we just requested:

~~~bash
[root@ocp4-bastion ~]# oc get pv |grep nfs
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                             STORAGECLASS   REASON   AGE
nfs-pv1   40Gi       RWO,RWX        Delete           Bound       default/rhel8-nfs                 nfs                     73m
nfs-pv2   40Gi       RWO,RWX        Delete           Available                                     nfs                     72m
~~~

Recall that when we setup the `PV` resources we specified the location and path of the NFS server that we wanted to utilise:

~~~basic
  nfs:
    path: /nfs/pv2
    server: 192.168.123.100
~~~

If we have a look on our NFS server we can make sure that it's using this correctly

> **NOTE**: In your environment, if your image was 'pv2' rather than 'pv1', adjust the commands to match your setup. 

~~~bash
[root@ocp4-bastion ~]# ls -l /nfs/pv1
total 10495876
-rw-rw----. 1 root root 40587440640 Jul 22 21:09 disk.img

[root@ocp4-bastion ~]# ls -l /nfs/pv1
total 10495876
-rw-rw----. 1 root root 40587440640 Jul 22 21:09 disk.img

[root@ocp4-bastion ~]# qemu-img info /nfs/pv1/disk.img
image: /nfs/pv1/disk.img
file format: raw
virtual size: 37.8 GiB (40587440640 bytes)
disk size: 10 GiB

[root@ocp4-bastion ~]# file /nfs/pv1/disk.img
/nfs/pv1/disk.img: DOS/MBR boot sector
~~~

We'll use this NFS-based RHEL8 image when we provision a virtual machine in one of the next steps.


## Hostpath Storage

Now let's create a second storage type based on `hostpath` storage. We'll follow a similar approach to setting this up, but we won't be using shared storage, so all of the data that we create on hostpath type volumes is essentially ephemeral, and resides on the local disk of the hypervisors (OpenShift worker's) themselves. As we're not using a pre-configured shared storage pool for this we need to ask OpenShift's `MachineConfigOperator` to do some work for us directly on our worker nodes.

Run the following from the command line - it will generate a new `MachineConfig` that the cluster will enact, recognising that we only match on the worker nodes (`machineconfiguration.openshift.io/role: worker`):

~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 50-set-selinux-for-hostpath-provisioner-worker
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 2.2.0
    systemd:
      units:
        - contents: |
            [Unit]
            Description=Set SELinux chcon for hostpath provisioner
            Before=kubelet.service

            [Service]
            Type=oneshot
            RemainAfterExit=yes
            ExecStartPre=-mkdir -p /var/hpvolumes
            ExecStart=/usr/bin/chcon -Rt container_file_t /var/hpvolumes

            [Install]
            WantedBy=multi-user.target
          enabled: true
          name: hostpath-provisioner.service
EOF

machineconfig.machineconfiguration.openshift.io/50-set-selinux-for-hostpath-provisioner-worker created
~~~

This deploys a new `systemd` unit file on the worker nodes to create a new directory at `/var/hpvolumes` and relabels it with the correct SELinux contexts at boot-time, ensuring that OpenShift can leverage that directory for local storage. We do this via a `MachineConfig` as the CoreOS machine is immutable. We should first start to witness OpenShift starting to drain the worker nodes and disable scheduling on them so the nodes can be rebooted safely:

~~~bash
[root@ocp4-bastion ~]# oc get nodes
NAME                           STATUS                     ROLES    AGE     VERSION
ocp4-master1.aio.example.com   Ready                      master   25h     v1.20.0+87cc9a4
ocp4-master2.aio.example.com   Ready                      master   25h     v1.20.0+87cc9a4
ocp4-master3.aio.example.com   Ready                      master   25h     v1.20.0+87cc9a4
ocp4-worker1.aio.example.com   Ready                      worker   25h     v1.20.0+87cc9a4
ocp4-worker2.aio.example.com   Ready,SchedulingDisabled   worker   25h     v1.20.0+87cc9a4
ocp4-worker3.aio.example.com   Ready                      worker   7h23m   v1.20.0+87cc9a4
~~~

Now wait for the following command to return `True` as it indicates when the `MachineConfigPool`'s worker has been updated with the latest `MachineConfig` requested: 

~~~bash
[root@ocp4-bastion ~]# oc get machineconfigpool worker -o=jsonpath="{.status.conditions[?(@.type=='Updated')].status}{\"\n\"}"
False

[root@ocp4-bastion ~]# oc get machineconfigpool worker -o=jsonpath="{.status.conditions[?(@.type=='Updated')].status}{\"\n\"}"
True
~~~

Now we can set the HostPathProvisioner configuration itself, i.e. telling the operator what path to actually use - the systemd file we just applied merely ensures that the directory is present and has the correct SELinux labels applied to it:

~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: hostpathprovisioner.kubevirt.io/v1beta1
kind: HostPathProvisioner
metadata:
  name: hostpath-provisioner
spec:
  imagePullPolicy: IfNotPresent
  pathConfig:
    path: "/var/hpvolumes"
    useNamingPrefix: false
EOF
hostpathprovisioner.hostpathprovisioner.kubevirt.io/hostpath-provisioner created
~~~

When you've applied this config, an additional pod will be spawned on each of the worker nodes; this pod is responsible for managing the hostpath access on the respective host; note the shorter age (42s in the example below):

~~~bash
[root@ocp4-bastion ~]# oc get pods -n openshift-cnv | grep hostpath
hostpath-provisioner-df5b9                            1/1     Running   0          25s
hostpath-provisioner-operator-695bf6fbd6-46ks5        1/1     Running   0          11m
hostpath-provisioner-wnbvz                            1/1     Running   0          25s
hostpath-provisioner-xjvzp                            1/1     Running   0          25s
~~~

We're now ready to configure a new `StorageClass` for the HostPath based storage:

~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath-provisioner
provisioner: kubevirt.io/hostpath-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
storageclass.storage.k8s.io/hostpath-provisioner created
~~~

Confirm the storageclass was created:

~~~bash
[root@ocp4-bastion ~]# oc get sc hostpath-provisioner
NAME                   PROVISIONER                        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
hostpath-provisioner   kubevirt.io/hostpath-provisioner   Delete          WaitForFirstConsumer   false                  31s
~~~

You'll note that this storage class **does** have a provisioner (as opposed to the previous use of `kubernetes.io/no-provisioner`), and therefore it can create persistent volumes dynamically when a claim is submitted by the user, let's validate that by creating a new hostpath based PVC and checking that it creates the associated PV:


~~~bash
[root@ocp4-bastion ~]# cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "rhel8-hostpath"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "http://192.168.123.100:81/rhel8-kvm.img"
    volume.kubernetes.io/selected-node: ocp4-worker3.aio.example.com                        
spec:
  volumeMode: Filesystem
  storageClassName: hostpath-provisioner
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 40Gi
EOF

persistentvolumeclaim/rhel8-hostpath created
~~~

We use CDI to ensure the volume we're requesting uses a RHEL8 image that we're hosting on a pre-configured web-server on the bastion node, so we expect the importer image to run again:

~~~bash
[root@ocp4-bastion ~]# oc get pods
NAME                      READY   STATUS    RESTARTS   AGE
importer-rhel8-hostpath   1/1     Running   0          19s
~~~

> **NOTE**: You can watch the output of this importer pod with `$ oc logs -f importer-rhel8-hostpath`.  Didn't see any pods? You likely just missed it. To be sure the PV was created continue to the next command.

Once that pod has finished let's check the status of the PV's:

~~~bash
[root@ocp4-bastion ~]# oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                           STORAGECLASS                  REASON   AGE
local-pv-6751e4c7                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-0-data-09jqjr   localblock                             17h
local-pv-759d6092                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-2-data-16d4zg   localblock                             17h
local-pv-911d74ba                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-1-data-0dw2b5   localblock                             17h
local-pv-9d4e1eb0                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-0-data-15rq28   localblock                             17h
local-pv-be875671                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-2-data-0d8spr   localblock                             17h
local-pv-e2c7f8a9                          100Gi      RWO            Delete           Bound    openshift-storage/ocs-deviceset-1-data-145ttc   localblock                             17h
pvc-44178021-c910-42c5-bd27-17b31ddc95bf   50Gi       RWO            Delete           Bound    openshift-storage/db-noobaa-db-0                ocs-storagecluster-ceph-rbd            17h
pvc-c056562a-52a0-4544-8ebd-35a846fa9f00   79Gi       RWO            Delete           Bound    default/rhel8-hostpath                          hostpath-provisioner                   7h3m
~~~

> **NOTE**: The capacity displayed above lists the available space on the host, not the actual size of the persistent volume when being used.


Let's look more closely at the PV's. Describe the new hostpath PV (noting that you'll need to adapt for the `uuid` in your environment):

~~~bash
[root@ocp4-bastion ~]# oc describe pv pvc-c056562a-52a0-4544-8ebd-35a846fa9f00
Name:              pvc-c056562a-52a0-4544-8ebd-35a846fa9f00
Labels:            <none>
Annotations:       hostPathProvisionerIdentity: kubevirt.io/hostpath-provisioner
                   kubevirt.io/provisionOnNode: ocp4-worker3.aio.example.com
                   pv.kubernetes.io/provisioned-by: kubevirt.io/hostpath-provisioner
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      hostpath-provisioner
Status:            Bound
Claim:             default/rhel8-hostpath
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          79Gi
Node Affinity:     
  Required Terms:  
    Term 0:        kubernetes.io/hostname in [ocp4-worker3.aio.example.com]
Message:           
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /var/hpvolumes/pvc-c056562a-52a0-4544-8ebd-35a846fa9f00
    HostPathType:  
Events:            <none>
~~~

There's a few important details here worth noting, namely the `kubevirt.io/provisionOnNode` annotation, and the path of the volume on that node. In the example above you can see that the volume was provisioned on *ocp4-worker3.aio.example.com*, the third of our three worker nodes (in your environment it may have been scheduled onto the second worker). 

Let's look more closely to verify that this truly has been created for us on the designated worker.

> **NOTE**: You may have to substitute `ocp4-worker3` with `ocp4-worker[1-2]` if your hostpath volume was scheduled to worker[1-2]. You'll need to also match the UUID to the one that was generated by your PVC. 

~~~bash
[root@ocp4-bastion ~]# oc debug node/ocp4-worker3.aio.example.com
Starting pod/ocp4-worker3aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.106
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-4.4# ls -l /var/hpvolumes/
total 0
drwxrwxrwx. 2 root root 22 Aug 11 10:41 pvc-c056562a-52a0-4544-8ebd-35a846fa9f00
sh-4.4# ls -l /var/hpvolumes/pvc-c056562a-52a0-4544-8ebd-35a846fa9f00
total 10485764
-rw-rw----. 1 root root 40587440640 Aug 11 10:42 disk.img
sh-4.4# file /var/hpvolumes/pvc-c056562a-52a0-4544-8ebd-35a846fa9f00/disk.img 
/var/hpvolumes/pvc-c056562a-52a0-4544-8ebd-35a846fa9f00/disk.img: DOS/MBR boot sector
sh-4.4# exit
exit
sh-4.4# exit
exit

Removing debug pod ...
~~~

Make sure that you've executed the two `exit` commands above - we need to make sure that you're back to the right shell before continuing, and aren't still inside of the debug pod.
