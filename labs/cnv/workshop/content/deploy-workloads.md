
## Deploy Virtual Machine using CLI

Now let's bring all these configurations together and actually launch some workloads!

> **NOTE**: We're calling most of the resources "RHEL 8" here, regardless of whether you're actually using CentOS 8 as your base image - it won't impact anything for our purposes here.

To begin with let's use the OpenShift Data Foundation volume we created earlier to launch some VMs. We are going to create a machine called `rhel8-server-ocs`. As you'll recall we have created a PVC called `rhel8-ocs` that was created using the CDI utility with a CentOS 8 base image. To connect the machine to the network we will utilise the `NetworkAttachmentDefinition` we created for the underlying host's third NIC (`enp3s0` via `br1`). This is the `tuning-bridge-fixed` interface which refers to that bridge created previously. It's also important to remember that OpenShift 4.x uses Multus as it's default networking CNI so we also ensure Multus knows about this `NetworkAttachmentDefinition`. Lastly we have set the `evictionStrategy` to `LiveMigrate` so that any request to move the instance will use this method. We will explore this in more depth in a later lab.

Let's apply a VM configuration via the CLI first:

```execute-1
cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    name.os.template.kubevirt.io/rhel8: Red Hat Enterprise Linux 8.0
  labels:
    app: rhel8-server-ocs
    kubevirt.io/os: rhel8
    os.template.kubevirt.io/rhel8: 'true'
    template.kubevirt.ui: openshift_rhel8-generic-small
    vm.kubevirt.io/template: openshift_rhel8-generic-small
    workload.template.kubevirt.io/generic: 'true'
  name: rhel8-server-ocs
spec:
  running: true
  template:
    metadata:
      labels:
        vm.kubevirt.io/name: rhel8-server-ocs
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
          - disk:
              bus: sata
            name: rhel8-ocs
          interfaces:
          - bridge: {}
            model: e1000
            name: tuning-bridge-fixed
        firmware:
          uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
        machine:
          type: q35
        resources:
          requests:
            memory: 512M
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
```

You should see VirtualMachine object is created.

~~~bash
virtualmachine.kubevirt.io/rhel8-server-ocs created
~~~

This starts to **schedule** the virtual machine across the available hypervisors, which we can see by viewing the VM and VMI objects:


```execute-1
oc get vm
```

This command will list the VirtualMachine objects:

~~~bash
NAME               AGE   STATUS     READY
rhel8-server-ocs   4s    Starting   False
~~~

Now execute following command:

```execute-1
oc get vmi
```

This command will list the VirtualMachineInstance objects:


~~~bash
NAME               AGE   PHASE     IP    NODENAME                       READY
rhel8-server-ocs   15s   Running         ocp4-worker3.%node-network-domain%   True
~~~

> **NOTE**: A `vm` object is the definition of the virtual machine, whereas a `vmi` is an instance of that virtual machine definition. In addition, you need to have the `qemu-guest-agent` installed in the guest for the IP address to show in this list, and it may take a minute or two to appear.

What you'll find is that OpenShift spawns a pod that manages the provisioning of the virtual machine in our environment, known as the `virt-launcher`:

```execute-1
oc get pods
```

Check whether `virt-launcher` is running:

~~~bash
NAME                                   READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-server-ocs-z5rmr   1/1     Running   0          3m5s
~~~

Then execute following to describe the details:

```copy
oc describe pod virt-launcher-rhel8-server-ocs-[podid]
```

This command will list pod details in yaml format:

~~~yaml
Name:         virt-launcher-rhel8-server-ocs-z5rmr
Namespace:    default
Priority:     0
Node:         ocp4-worker3.%node-network-domain%/192.168.123.106
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

```execute-1
oc get pods
```

See the pod is running

~~~bash
NAME                                   READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-server-ocs-z5rmr   1/1     Running   0          30h
~~~

Execute following to open a bash shell in the pod (replace the pod name):

```copy
oc exec -it virt-launcher-rhel8-server-ocs-z5rmr -- bash
```
And then you can run the usual virsh commands:


```execute-1
virsh list --all
```

Verify the process is running

~~~bash
 Id   Name                       State
------------------------------------------
 1    default_rhel8-server-ocs   running
~~~

We can also verify the storage attachment, which should be an RBD volume as it's come from OCS:

```execute-1
virsh domblklist default_rhel8-server-ocs
```

This command will list the block devices:

~~~bash
 Target   Source
--------------------------
 sda      /dev/rhel8-ocs
~~~

And execute the following to check block device:

```execute-1
lsblk /dev/rhel8-ocs 
```

This command will list information about specified block device:

~~~bash
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
rbd1     251:16   0   40G  0 disk 
└─rbd1p1 251:17   0  7.8G  0 part
~~~

And for networking:

```execute-1
virsh domiflist default_rhel8-server-ocs
```

This command will list information about network interfaces:

~~~bash
 Interface   Type     Source     Model   MAC
------------------------------------------------------------
 tap1        ethernet   -        e1000   02:7c:a4:00:00:00
~~~

But let's go a little deeper with the following command:

```execute-1
ip link | grep -A2 tap1
```

If we look at tap1 we'll see that it's part of a bridge called "*k6t-net1*":

~~~bash
6: tap1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master k6t-net1 state UP mode DEFAULT group default qlen 1000
    link/ether ca:e9:2f:35:2c:a1 brd ff:ff:ff:ff:ff:ff
~~~

That bridge device has an interface called "*net1@if29*" (yours may be slightly different):

```execute-1
ip link | grep -A2 k6t-net1
```

 You should see the details:

~~~bash
4: net1@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master k6t-net1 state UP mode DEFAULT group default 
    link/ether 02:7c:a4:12:0d:6f brd ff:ff:ff:ff:ff:ff link-netnsid 0
5: k6t-net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7c:a4:12:0d:6f brd ff:ff:ff:ff:ff:ff
6: tap1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master k6t-net1 state UP mode DEFAULT group default qlen 1000
    link/ether ca:e9:2f:35:2c:a1 brd ff:ff:ff:ff:ff:ff
~~~

That's showing that there's a bridge inside of the pod called "**k6t-net1**" with both the **"tap1"** (the device attached to the VM), and the **"net1@if29"** device being how the packets get out onto the bridge on the hypervisor (more shortly):

```execute-1
virsh dumpxml default_rhel8-server-ocs | grep -A8 "interface type"
```

This will show interface information in XML format:

~~~xml
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

Exit the shell before proceeding:

```execute-1
exit
```