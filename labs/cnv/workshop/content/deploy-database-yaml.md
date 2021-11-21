Now let's bring all these configurations together and actually launch some workloads!

> **NOTE**: We're calling most of the resources "RHEL 8" here, regardless of whether you're actually using CentOS 8 as your base image - it won't impact anything for our purposes here.

To begin with let's use the OpenShift Data Foundation volume we created earlier to launch some VMs. We are going to create a machine called `rhel8-server-ocs`. As you'll recall we have created a PVC called `rhel8-ocs` that was created using the CDI utility with a CentOS 8 base image. To connect the machine to the network we will utilise the `NetworkAttachmentDefinition` we created for the underlying host's third NIC (`enp3s0` via `br1`). This is the `tuning-bridge-fixed` interface which refers to that bridge created previously. It's also important to remember that OpenShift 4.x uses Multus as it's default networking CNI so we also ensure Multus knows about this `NetworkAttachmentDefinition`. Lastly we have set the `evictionStrategy` to `LiveMigrate` so that any request to move the instance will use this method. We will explore this in more depth in a later lab.

Let's apply a VM configuration via the CLI first:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    name.os.template.kubevirt.io/rhel8: Red Hat Enterprise Linux 8.0
  labels:
    flavor.template.kubevirt.io/small: "true"
    kubevirt.io/os: rhel8
    os.template.kubevirt.io/rhel8: "true"
    template.kubevirt.ui: openshift_rhel8-generic-large
    vm.kubevirt.io/template: rhel8-generic-small
    workload.template.kubevirt.io/generic: "true"
    app: rhel8-server-ocs
  name: rhel8-server-ocs
spec:
  running: true
  template:
    metadata:
      labels:
        vm.kubevirt.io/name: rhel8-server-ocs
    spec:
      domain:
        clock:
          timer:
            hpet:
              present: false
            hyperv: {}
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
          utc: {}
        devices:
          disks:
          - disk:
              bus: sata
            name: rhel8-ocs
          interfaces:
          - bridge: {}
            model: e1000
            name: tuning-bridge-fixed
        features:
          acpi: {}
          apic: {}
          hyperv:
            relaxed: {}
            spinlocks:
              spinlocks: 8191
            vapic: {}
        firmware:
          uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
        machine:
          type: q35
        resources:
          requests:
            memory: 2048M
      evictionStrategy: LiveMigrate
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
      - name: rhel8-ocs
        persistentVolumeClaim:
          claimName: rhel8-ocs
EOF

virtualmachine.kubevirt.io/rhel8-server-ocs created
~~~

This starts to **schedule** the virtual machine across the available hypervisors, which we can see by viewing the VM and VMI objects:

~~~bash
$ oc get vm
NAME               AGE   STATUS     READY
rhel8-server-ocs   4s    Starting   False

$ oc get vmi
NAME               AGE   PHASE     IP    NODENAME                       READY
rhel8-server-ocs   15s   Running         ocp4-worker3.aio.example.com   True
~~~

> **NOTE**: A `vm` object is the definition of the virtual machine, whereas a `vmi` is an instance of that virtual machine definition. In addition, you need to have the `qemu-guest-agent` installed in the guest for the IP address to show in this list, and it may take a minute or two to appear.

What you'll find is that OpenShift spawns a pod that manages the provisioning of the virtual machine in our environment, known as the `virt-launcher`:

~~~bash
$ oc get pods
NAME                                   READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-server-ocs-z5rmr   1/1     Running   0          3m5s

$ oc describe pod virt-launcher-rhel8-server-ocs-z5rmr
Name:         virt-launcher-rhel8-server-ocs-z5rmr
Namespace:    default
Priority:     0
Node:         ocp4-worker3.aio.example.com/192.168.123.106
Start Time:   Mon, 08 Nov 2021 13:47:23 +0000
Labels:       kubevirt.io=virt-launcher
              kubevirt.io/created-by=8d58ce9f-61bd-4c5f-82d1-15dbc9c3897e
              vm.kubevirt.io/name=rhel8-server-ocs
Annotations:  k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.129.2.16"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/tuning-bridge-fixed",
                    "interface": "net1",
                    "mac": "02:7c:a4:00:00:00",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks: [{"interface":"net1","mac":"02:7c:a4:00:00:00","name":"tuning-bridge-fixed","namespace":"default"}]
              k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.129.2.16"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/tuning-bridge-fixed",
                    "interface": "net1",
                    "mac": "02:7c:a4:00:00:00",
                    "dns": {}
                }]
              kubevirt.io/domain: rhel8-server-ocs
Status:       Running
(...)
~~~

If you look into this launcher pod, you'll see that it has the same typical libvirt functionality as we've come to expect with existing Red Hat virtualisation products like RHV and/or OpenStack. First get a shell on the pod that's operating our virtual machine, recalling that each VM has a `virt-launcher` pod associated to it:

~~~bash
$ oc get pods
NAME                                   READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-server-ocs-z5rmr   1/1     Running   0          30h

$ oc exec -it virt-launcher-rhel8-server-ocs-z5rmr bash
(...)
~~~

And then you can run the usual virsh commands:

~~~bash
[root@rhel8-server-ocs /]# virsh list --all
 Id   Name                       State
------------------------------------------
 1    default_rhel8-server-ocs   running
~~~

We can also verify the storage attachment, which should be an RBD volume as it's come from OCS:

~~~bash
[root@rhel8-server-ocs /]# virsh domblklist default_rhel8-server-ocs
 Target   Source
--------------------------
 sda      /dev/rhel8-ocs
 
[root@rhel8-server-ocs /]# lsblk /dev/rhel8-ocs 
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
rbd1     251:16   0   40G  0 disk 
└─rbd1p1 251:17   0  7.8G  0 part
~~~

And for networking:

~~~bash
[root@rhel8-server-nfs /]# virsh domiflist default_rhel8-server-ocs
 Interface   Type     Source     Model   MAC
------------------------------------------------------------
 tap1        ethernet   -        e1000   02:7c:a4:00:00:00
~~~

But let's go a little deeper; if we look at tap1 we'll see that it's part of a bridge called "*k6t-net1*":

~~~bash
[root@rhel8-server-ocs /]# ip link | grep -A2 tap1
6: tap1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master k6t-net1 state UP mode DEFAULT group default qlen 1000
    link/ether ca:e9:2f:35:2c:a1 brd ff:ff:ff:ff:ff:ff
~~~

That bridge device has an interface called "*net1@if29*" (yours may be slightly different):

~~~bash
[root@rhel8-server-ocs /]# ip link | grep -A2 k6t-net1
4: net1@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master k6t-net1 state UP mode DEFAULT group default 
    link/ether 02:7c:a4:12:0d:6f brd ff:ff:ff:ff:ff:ff link-netnsid 0
5: k6t-net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7c:a4:12:0d:6f brd ff:ff:ff:ff:ff:ff
6: tap1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master k6t-net1 state UP mode DEFAULT group default qlen 1000
    link/ether ca:e9:2f:35:2c:a1 brd ff:ff:ff:ff:ff:ff
~~~

That's showing that there's a bridge inside of the pod called "**k6t-net1**", with both the **"tap1"** (the device attached to the VM), and the **"net1@if29"** device being how the packets get out onto the bridge on the hypervisor (more shortly):

~~~bash
[root@rhel8-server-ocs /]# virsh dumpxml default_rhel8-server-ocs | grep -A8 "interface type"
    <interface type='ethernet'>
      <mac address='02:7c:a4:00:00:00'/>
      <target dev='tap1' managed='no'/>
      <model type='e1000'/>
      <mtu size='1500'/>
      <alias name='ua-tuning-bridge-fixed'/>
      <rom enabled='no'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x01' function='0x0'/>
    </interface>
~~~

Exit the shell before proceeding (the `oc whoami` here just makes sure you're in the right place):

~~~bash
[root@rhel8-server-ocs /]# exit
exit

$ oc whoami
system:serviceaccount:workbook:cnv
~~~

Now, how is this plugged on the underlying host? 

The key to this is the **"net1@if29"** device (it may be slightly different in your environment); this is one half of a **"veth pair"** that allows network traffic to be bridged between network namespaces, which is exactly how containers segregate their network traffic between each other on a container host. In this example the **"cnv-bridge"** is being used to connect the bridge for the virtual machine (**"k6t-net1"**) out to the bridge on the underlying host (**"br1"**), via a veth pair. The other side of the veth pair can be discovered as follows. First find the host of our virtual machine:

~~~bash
$ oc get vmi
NAME               AGE   PHASE     IP               NODENAME                       READY
rhel8-server-ocs   30h   Running   192.168.123.64   ocp4-worker3.aio.example.com   True
~~~

Then connect to it and track back the link - here you'll need to adjust the commands below - if your veth pair on the pod side was **"net1@if29"** then the **ifindex** in the command below will be **"29"**, if it was **"net1@if5"** then **"ifindex"** will be **"5"** and so on...

To do this we need to get to the worker running our virtual machine, and we can use the `oc debug node` function to do this (adjust to suit the host of your virtual machine from the previous command):

~~~bash
$ oc debug node/ocp4-worker3.aio.example.com
Starting pod/ocp4-worker3aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.106
If you don't see a command prompt, try pressing enter.

sh-4.4# chroot /host
sh-4.4# export ifindex=29
sh-4.4# ip -o link | grep ^$ifindex: | sed -n -e 's/.*\(veth[[:alnum:]]*@if[[:digit:]]*\).*/\1/p'
Error: Peer netns reference is invalid.
veth9d567769@if4
~~~
Therefore, the other side of the link, in the example above is **"veth9d567769@if4"**. You can then see that this is attached to **"br1"** as follows-

~~~bash
sh-4.4# ip link show veth9d567769     
29: veth9d567769@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br1 state UP mode DEFAULT group default 
Error: Peer netns reference is invalid.
    link/ether 16:6b:ba:90:da:87 brd ff:ff:ff:ff:ff:ff link-netns c3dde5db-32f3-47a7-b7a8-3705dd17f8e1
~~~

Note the "**master br1**" in the above output.

Or visually represented:

<img src="img/veth-pair.png" />

Exit the debug shell(s) before proceeding:

~~~bash
sh4.4# exit
exit
sh4.2# exit
exit

Removing debug pod ...

$ oc whoami
system:serviceaccount:workbook:cnv

$ oc project default
Already on project "default" on server "https://172.30.0.1:443".
~~~

Now that we have the OCS instance running, let's do the same for the **hostpath** setup we created. Let's leverage the hostpath PVC that we created in a previous step - this is essentially the same as our OCS-based VM instance, except we reference the `rhel8-hostpath` PVC instead:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    name.os.template.kubevirt.io/rhel8: Red Hat Enterprise Linux 8.0
  labels:
    flavor.template.kubevirt.io/small: "tiny"
    kubevirt.io/os: rhel8
    os.template.kubevirt.io/rhel8: "true"
    template.kubevirt.ui: openshift_rhel8-generic-tiny
    vm.kubevirt.io/template: tiny
    workload.template.kubevirt.io/generic: "true"
    app: rhel8-server-hostpath
  name: rhel8-server-hostpath
spec:
  running: true
  template:
    metadata:
      labels:
        vm.kubevirt.io/name: rhel8-server-hostpath
    spec:
      domain:
        clock:
          timer:
            hpet:
              present: false
            hyperv: {}
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
          utc: {}
        devices:
          disks:
          - disk:
              bus: sata
            name: rhel8-hostpath
          interfaces:
          - bridge: {}
            model: e1000
            name: tuning-bridge-fixed
        features:
          acpi: {}
          apic: {}
          hyperv:
            relaxed: {}
            spinlocks:
              spinlocks: 8191
            vapic: {}
        firmware:
          uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
        machine:
          type: q35
        resources:
          requests:
            memory: 1024M
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
      - name: rhel8-hostpath
        persistentVolumeClaim:
          claimName: rhel8-hostpath
EOF

virtualmachine.kubevirt.io/rhel8-server-hostpath created 
~~~

As before we can see the launcher pod built and run:

~~~bash
$ oc get pods
NAME                                        READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-server-hostpath-ddmjm   0/1     Pending   0          4s
virt-launcher-rhel8-server-ocs-z5rmr        1/1     Running   0          30h
~~~

And after a few seconds, this should launch...

~~~bash
$ oc get vmi
NAME                    AGE   PHASE     IP               NODENAME                       READY
rhel8-server-hostpath   71s   Running   192.168.123.65   ocp4-worker2.aio.example.com   True
rhel8-server-ocs        30h   Running   192.168.123.64   ocp4-worker3.aio.example.com   True
~~~

And looking deeper we can see the hostpath claim we explored earlier being utilised, note the `Mounts` section for where, inside the pod, the `rhel8-hostpath` PVC is attached, and then below the PVC name:

~~~bash
$ oc describe pod/virt-launcher-rhel8-server-hostpath-9sqxm
Name:         virt-launcher-rhel8-server-hostpath-9sqxm
Namespace:    default
Priority:     0
Node:         ocp4-worker2.aio.example.com/192.168.123.105
Start Time:   Tue, 09 Nov 2021 20:33:22 +0000
Labels:       kubevirt.io=virt-launcher
              kubevirt.io/created-by=7a19cb48-18bb-4d16-a7eb-4f7811675c17
              vm.kubevirt.io/name=rhel8-server-hostpath
(...)    
    Mounts:
      /var/run/kubevirt from public (rw)
      /var/run/kubevirt-ephemeral-disks from ephemeral-disks (rw)
      /var/run/kubevirt-private from private (rw)
      /var/run/kubevirt-private/vmi-disks/rhel8-hostpath from rhel8-hostpath (rw)
      /var/run/kubevirt/container-disks from container-disks (rw)
      /var/run/kubevirt/hotplug-disks from hotplug-disks (rw)
      /var/run/kubevirt/sockets from sockets (rw)
      /var/run/libvirt from libvirt-runtime (rw)
(...)
Volumes:
  rhel8-hostpath:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  rhel8-hostpath
    ReadOnly:   false
(...)
~~~

If we take a peek inside of that pod we can dig down further, you'll notice that it's largely the same output as the OCS step above, but the mount point is obviously not over OCS:

~~~bash
$ oc exec -it virt-launcher-rhel8-server-hostpath-9sqxm bash
(...)

[root@rhel8-server-hostpath /]# virsh domblklist default_rhel8-server-hostpath
 Target   Source
-----------------------------------------------------------------------
 sda      /var/run/kubevirt-private/vmi-disks/rhel8-hostpath/disk.img
~~~

The problem here is that if you try and run the `mount` command to see how this is attached, it will only show the whole filesystem being mounted into the pod:

~~~bash
[root@rhel8-server-hostpath /]# mount | grep rhel8
/dev/vda4 on /run/kubevirt-private/vmi-disks/rhel8-hostpath type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota)
~~~

However, we can see how this has been mapped in by Kubernetes by looking at the pod configuration on the host. First, recall which host your virtual machine is running on, and get the name of the launcher pod:

~~~bash
[root@rhel8-server-hostpath /]# exit
logout

$ oc get vmi/rhel8-server-hostpath
NAME                    AGE     PHASE     IP               NODENAME                       READY
rhel8-server-hostpath   6m43s   Running   192.168.123.65   ocp4-worker2.aio.example.com   True

$ oc get pods | awk '/hostpath/ {print $1;}'
virt-launcher-rhel8-server-hostpath-9sqxm
~~~

> **NOTE**: In your environment, your virtual machine may be running on worker1, simply adjust the following instructions to suit your configuration.

Next, we need to get the container ID from the pod:

~~~bash
$ oc describe pod virt-launcher-rhel8-server-hostpath-9sqxm | awk -F// '/Container ID/ {print $2;}'
ba06cc69e0dbe376dfa0c8c72f0ab5513f31ab9a7803dd0102e858c94df55744
~~~

> **NOTE**: Make a note (copy it) of this container ID as we'll use it in the next step

Now we can check on the worker node itself, remembering to adjust these commands for the worker that your hostpath based VM is running on, and the container ID from the above:

~~~bash
$ oc describe pod virt-launcher-rhel8-server-hostpath-9sqxm | grep "Node:"
Node:         ocp4-worker2.aio.example.com/192.168.123.105

$ oc debug node/ocp4-worker2.aio.example.com
Starting pod/ocp4-worker2aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.105
If you don't see a command prompt, try pressing enter.

sh-4.4# chroot /host
sh-4.4# crictl inspect ba06cc69e0dbe376dfa0c8c72f0ab5513f31ab9a7803dd0102e858c94df55744 | grep -A4 rhel8-hostpath

        "containerPath": "/var/run/kubevirt-private/vmi-disks/rhel8-hostpath",
        "hostPath": "/var/hpvolumes/pvc-0f3ff50d-12b1-4ad6-8beb-be742a6e674a",
        "propagation": "PROPAGATION_PRIVATE",
        "readonly": false,
        "selinuxRelabel": false
(...)
~~~

Here you can see that the container has been configured to have a `hostPath` from `/var/hpvolumes` mapped to the expected path inside of the container where the libvirt definition is pointing to. Don't forget to exit (twice) before proceeding:

~~~bash
sh4.4# exit
exit
sh4.2# exit
exit

Removing debug pod ...

$ oc whoami
system:serviceaccount:workbook:cnv

[~] $ oc project default
Already on project "default" on server "https://172.30.0.1:443".
~~~

That's it for deploying basic workloads - we've deployed a VM on-top of OCS and one on-top of hostpath.
