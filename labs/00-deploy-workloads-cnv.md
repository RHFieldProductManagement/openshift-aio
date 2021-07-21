# **Running Workloads in the Environment**

REWORK STILL REQUIRED

Let's get into actually testing some workloads on our environment... we've spent all of this time building it up but we haven't even proven that it's working properly yet! In this section we're going to be deploying some pods as well as deploying a VM using OpenShift Virtualization.

## Deploying a Virtual Machine

In the previous deploy OpenShift Virtualization lab we installed the OpenShift Virtualization operator and created a virtualization cluster.  We also configured an external bridge so that our virtual machines can be connected to the outside network.  

We are now at the point where we can launch a virtual machine!  To do this we will use the following YAML file below, recalling that as with all things OpenShift, we can deploy virtual machines in the same way as we would standard containerised pods.  Lets go ahead and create the file:

~~~bash
[lab-user@provision scripts]$ cat << EOF > ~/virtualmachine-fedora.yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/silverblue32: Fedora 31 or higher
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachines/fedora
  resourceVersion: '65643'
  name: fedora
  uid: 24c12216-ba66-49ec-beae-414ea4e7c06a
  creationTimestamp: '2020-10-17T21:50:00Z'
  generation: 4
  managedFields:
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            .: {}
            'f:name.os.template.kubevirt.io/silverblue32': {}
          'f:labels':
            'f:os.template.kubevirt.io/silverblue32': {}
            'f:vm.kubevirt.io/template.version': {}
            'f:vm.kubevirt.io/template.namespace': {}
            'f:flavor.template.kubevirt.io/medium': {}
            'f:app': {}
            .: {}
            'f:workload.template.kubevirt.io/desktop': {}
            'f:vm.kubevirt.io/template.revision': {}
            'f:vm.kubevirt.io/template': {}
        'f:spec':
          .: {}
          'f:dataVolumeTemplates': {}
          'f:running': {}
          'f:template':
            .: {}
            'f:metadata':
              .: {}
              'f:labels':
                .: {}
                'f:flavor.template.kubevirt.io/medium': {}
                'f:kubevirt.io/domain': {}
                'f:kubevirt.io/size': {}
                'f:os.template.kubevirt.io/silverblue32': {}
                'f:vm.kubevirt.io/name': {}
                'f:workload.template.kubevirt.io/desktop': {}
            'f:spec':
              .: {}
              'f:domain':
                .: {}
                'f:cpu':
                  .: {}
                  'f:cores': {}
                  'f:sockets': {}
                  'f:threads': {}
                'f:devices':
                  .: {}
                  'f:autoattachPodInterface': {}
                  'f:disks': {}
                  'f:inputs': {}
                  'f:interfaces': {}
                  'f:networkInterfaceMultiqueue': {}
                  'f:rng': {}
                'f:machine':
                  .: {}
                  'f:type': {}
                'f:resources':
                  .: {}
                  'f:requests':
                    .: {}
                    'f:memory': {}
              'f:evictionStrategy': {}
              'f:hostname': {}
              'f:networks': {}
              'f:terminationGracePeriodSeconds': {}
              'f:volumes': {}
      manager: Mozilla
      operation: Update
      time: '2020-10-17T21:50:00Z'
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            'f:kubevirt.io/latest-observed-api-version': {}
            'f:kubevirt.io/storage-observed-api-version': {}
        'f:status':
          .: {}
          'f:conditions': {}
          'f:created': {}
          'f:ready': {}
      manager: virt-controller
      operation: Update
      time: '2020-10-17T21:52:54Z'
  namespace: default
  labels:
    app: fedora
    flavor.template.kubevirt.io/medium: 'true'
    os.template.kubevirt.io/silverblue32: 'true'
    vm.kubevirt.io/template: fedora-desktop-medium-v0.11.3
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.11.3
    workload.template.kubevirt.io/desktop: 'true'
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1alpha1
      kind: DataVolume
      metadata:
        creationTimestamp: null
        name: fedora-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 10Gi
          storageClassName: ocs-storagecluster-ceph-rbd
          volumeMode: Block
        source:
          http:
            url: >-
              https://download.fedoraproject.org/pub/fedora/linux/releases/32/Cloud/x86_64/images/Fedora-Cloud-Base-32-1.6.x86_64.raw.xz
      status: {}
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/medium: 'true'
        kubevirt.io/domain: fedora
        kubevirt.io/size: medium
        os.template.kubevirt.io/silverblue32: 'true'
        vm.kubevirt.io/name: fedora
        workload.template.kubevirt.io/desktop: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          autoattachPodInterface: false
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          inputs:
            - bus: virtio
              name: tablet
              type: tablet
          interfaces:
            - bridge: {}
              model: virtio
              name: nic-0
          networkInterfaceMultiqueue: true
          rng: {}
        machine:
          type: pc-q35-rhel8.2.0
        resources:
          requests:
            memory: 4Gi
      evictionStrategy: LiveMigrate
      hostname: fedora32
      networks:
        - multus:
            networkName: brext
          name: nic-0
      terminationGracePeriodSeconds: 180
      volumes:
        - dataVolume:
            name: fedora-rootdisk
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              name: default
              hostname: fedora32
              password: redhat
              chpasswd: {expire: False}
          name: cloudinitdisk
status:
  conditions:
    - lastProbeTime: null
      lastTransitionTime: '2020-10-17T21:52:54Z'
      status: 'True'
      type: Ready
  created: true
  ready: true
EOF
~~~

Let's take a look at some important parts of this file!

We can tell OpenShift Virtualization to use some preset defaults, much like with Red Hat Virtualization. In the below section, among other things, we set a flavor of "medium" which provides some basic. pre-defined resource settings for things like CPU and memory, as well as the "workload" type to "desktop". This also prevides preset configuration to define resources suitable for a desktop machine as opposed to a server.

~~~bash
  labels:
    app: fedora
    flavor.template.kubevirt.io/medium: 'true'
    os.template.kubevirt.io/silverblue32: 'true'
    vm.kubevirt.io/template: fedora-desktop-medium-v0.11.3
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.11.3
    workload.template.kubevirt.io/desktop: 'true'
~~~

In this next section we are utilizing the storage environment we built in previous labs! In this case we are creating the root disk of our VM to be created as a PVC using our OpenShift Container Storage install. We do this by setting the **storageClassName** to **ocs-storagecluster-ceph-rbd** allowing us to access the Ceph install. We can even set the access mode to ReadWriteMany meaning this VM can be live migrated easily. Finally, we source URL to tell OpenShift Virtualization where to get an OS image to load into the PVC:


~~~bash
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1alpha1
      kind: DataVolume
      metadata:
        creationTimestamp: null
        name: fedora-rootdisk
      spec:
        pvc:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 10Gi
          storageClassName: ocs-storagecluster-ceph-rbd
          volumeMode: Block
        source:
          http:
            url: >-
              https://download.fedoraproject.org/pub/fedora/linux/releases/32/Cloud/x86_64/images/Fedora-Cloud-Base-32-1.6.x86_64.raw.xz
      status: {}
~~~

The storage info about works with the snippet below. In that, we create a dataVolume utilising the PVC containing the pre-loaded Operating System we download. When we boot the VM it will use that OS!

~~~bash
      volumes:
        - dataVolume:
            name: fedora-rootdisk
          name: rootdisk
~~~

Another neat thing we can do with OpenShift Virtualization is use cloud-config. Cloud-config allows us to set important funtionality into a metadata service that is read and actioned when the VM is first booted. In the case below, we are setting the new VMs hostname to "**fedora32**" and it's root password to "**redhat**".

~~~bash
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              name: default
              hostname: fedora32
              password: redhat
              chpasswd: {expire: False}
          name: cloudinitdisk
~~~

Finally let's call out two snippets that set up our networking. In this first section we define the first (nic-0) interface on the machine:


~~~bash
          interfaces:
            - bridge: {}
              model: virtio
              name: nic-0
~~~

We are then able to use the the following section to instruct OpenShift to utilise its default Container Network Interface (CNI) network provider, **multus**, to connect that NIC into the **brext** bridge we created in the previous labs. This ensure the VM is exposed to the external network and not just the internal OpenStack SDN (like pods are).


~~~bash
      networks:
        - multus:
            networkName: brext
          name: nic-0
~~~

Now we can go ahead and create the virtual machine:

~~~bash
[lab-user@provision scripts]$ oc create -f ~/virtualmachine-fedora.yaml 
virtualmachine.kubevirt.io/fedora created
~~~

If we run a oc get pods for the default namespace we will see an importer container.  Once it is running we can watch the logs:

~~~bash
[lab-user@provision ~]$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
importer-fedora-rootdisk   0/1     Pending   0          2s
[lab-user@provision ~]$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
importer-fedora-rootdisk   1/1     Running   0          18s
~~~

You can following the image importing process container which is pulling in the fedora image from the URL we had inside the *virtualmachine-fedora.yaml* file:

~~~bash
[lab-user@provision ~]$ oc logs importer-fedora-rootdisk -f
I1018 10:14:55.118650       1 importer.go:51] Starting importer
I1018 10:14:55.119845       1 importer.go:112] begin import process
I1018 10:14:55.722075       1 data-processor.go:277] Calculating available size
I1018 10:14:55.722969       1 data-processor.go:285] Checking out block volume size.
I1018 10:14:55.722979       1 data-processor.go:297] Request image size not empty.
I1018 10:14:55.722987       1 data-processor.go:302] Target size 10Gi.
I1018 10:14:55.825077       1 data-processor.go:206] New phase: TransferDataFile
I1018 10:14:55.826012       1 util.go:161] Writing data...
I1018 10:14:56.825462       1 prometheus.go:69] 0.14
I1018 10:14:57.825748       1 prometheus.go:69] 0.15
I1018 10:14:58.826019       1 prometheus.go:69] 1.00
I1018 10:14:59.826160       1 prometheus.go:69] 2.31
I1018 10:15:00.826360       1 prometheus.go:69] 3.77
I1018 10:15:01.826512       1 prometheus.go:69] 4.47
I1018 10:15:02.826838       1 prometheus.go:69] 5.33
I1018 10:15:03.827362       1 prometheus.go:69] 5.50
I1018 10:15:04.827521       1 prometheus.go:69] 5.75
I1018 10:15:05.827674       1 prometheus.go:69] 6.01
I1018 10:15:06.827881       1 prometheus.go:69] 6.03
I1018 10:15:07.828369       1 prometheus.go:69] 6.81
I1018 10:15:08.828523       1 prometheus.go:69] 7.84
I1018 10:15:09.828691       1 prometheus.go:69] 9.02
(...)
I1018 10:17:07.871219       1 prometheus.go:69] 100.00
I1018 10:17:08.872312       1 prometheus.go:69] 100.00
I1018 10:17:10.215935       1 data-processor.go:206] New phase: Resize
I1018 10:17:10.217033       1 data-processor.go:206] New phase: Complete
I1018 10:17:10.217151       1 importer.go:175] Import complete
~~~

If you get an error such as: 

~~~bash
[lab-user@provision scripts]$ oc logs importer-fedora-rootdisk -f
Error from server (BadRequest): container "importer" in pod "importer-fedora-rootdisk" is waiting to start: ContainerCreating
~~~

You just ran the `oc logs` command before the container has started. Try it again and it should work.

Once the importer process has completed the virtual machine will then begin to startup and go to a running state. To interact with the VM from the CLI we can install a special tool provied with OpenShift Virtualiation called `virtctl`. This CLI is like `oc` or `kubectl` but for working with VMs. Let's install virtctl before we proceed:

~~~bash
[lab-user@provision scripts]$ sudo dnf -y install kubevirt-virtctl
(...)                                                                                                           6.7 kB/s | 2.3 kB     00:00
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                           Architecture                            Version                                           Repository                                                       Size
===================================================================================================================================================================================================================
Installing:
 kubevirt-virtctl                                  x86_64                                  0.26.1-15.el8                                     cnv-2.3-for-rhel-8-x86_64-rpms                                  8.1 M

(...)
Installed:
  kubevirt-virtctl-0.26.1-15.el8.x86_64                                                                                                                                                                            

Complete!
~~~

Now use `virtctl` to interact with the virtual machine:

~~~bash
[lab-user@provision ~]$ virtctl -h
virtctl controls virtual machine related operations on your kubernetes cluster.

Available Commands:
  console      Connect to a console of a virtual machine instance.
  expose       Expose a virtual machine instance, virtual machine, or virtual machine instance replica set as a new service.
  help         Help about any command
  image-upload Upload a VM image to a DataVolume/PersistentVolumeClaim.
  migrate      Migrate a virtual machine.
  pause        Pause a virtual machine
  restart      Restart a virtual machine.
  start        Start a virtual machine.
  stop         Stop a virtual machine.
  unpause      Unpause a virtual machine
  version      Print the client and server version information.
  vnc          Open a vnc connection to a virtual machine instance.

Use "virtctl <command> --help" for more information about a given command.
Use "virtctl options" for a list of global command-line options (applies to all commands).
~~~

Let's go ahead and see what the status of our virtual machine by connecting to the console (you may need to hit enter a couple of times):

~~~bash
[lab-user@provision ~]$ virtctl console fedora
Successfully connected to fedora console. The escape sequence is ^]

fedora32 login: 
~~~

As defined by the cloud-config information in our virtual machine YAML file we used the password should be set to **redhat** for the **fedora** user.  Lets login:

~~~bash
fedora32 login: fedora
Password: 
[fedora@fedora32 ~]$ cat /etc/fedora-release
Fedora release 32 (Thirty Two)
~~~

Now lets see if we got a 172.22.0.0/24 network address.  If you reference back to the virtual machines YAML file you will notice we used the brext bridge interface as the interface our virtual machine should be connected to.  Thus we should have a 172.22.0.0/24 network address and access outside:

~~~bash
[fedora@fedora32 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 46:84:6a:fb:ed:ee brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 172.22.0.29/24 brd 172.22.0.255 scope global dynamic noprefixroute eth0
       valid_lft 3176sec preferred_lft 3176sec
    inet6 fe80::4484:6aff:fefb:edee/64 scope link 
       valid_lft forever preferred_lft forever
~~~

Lets see if we can ping the gateway of 172.22.0.1:

~~~bash
[fedora@fedora32 ~]$ ping 172.22.0.1
PING 172.22.0.1 (172.22.0.1) 56(84) bytes of data.
64 bytes from 172.22.0.1: icmp_seq=1 ttl=64 time=1.63 ms
64 bytes from 172.22.0.1: icmp_seq=2 ttl=64 time=0.800 ms
64 bytes from 172.22.0.1: icmp_seq=3 ttl=64 time=0.848 ms
~~~

Looks like we have successful external network connectivity!

Now lets escape (using `ctrl-]`) out of the console session and connect to the fedora instance via ssh from the provisioning host:

~~~bash
fedora32 login: [lab-user@provision ocp]$ ssh fedora@172.22.0.29
The authenticity of host '172.22.0.29 (172.22.0.29)' can't be established.
ECDSA key fingerprint is SHA256:KAqdT3NUhrat+tfEV8e9V/hvWL8v5CQVsbULKxCLZp8.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.22.0.29' (ECDSA) to the list of known hosts.
fedora@172.22.0.29's password: 
Last login: Sun Oct 18 15:31:04 2020
[systemd]
Failed Units: 1
  dnf-makecache.service
[fedora@fedora32 ~]$ 
[fedora@fedora32 ~]$ 
[fedora@fedora32 ~]$ exit
~~~

Now our external network in this lab is actually the provisioning network we created a bridge on so it really does not have full functional connectivity to the internet.  However we can add a few bits to confirm that indeed if our bridge was a truely routable network we would be able to access the internet.  To simulate this we need to add the following firewalld rule on the provisioning node:

~~~bash
[lab-user@provision ~]$ sudo firewall-cmd --direct --add-rule ipv4 nat POSTROUTING 0 -o baremetal -j MASQUERADE
success
~~~

Then back on our fedora virtual machine we need to add the following default gateway which is the IP of the provisioning nodes interface:

~~~bash
[lab-user@provision scripts]$ ssh fedora@172.22.0.29
[fedora@fedora32 ~]$ sudo route add default gw 172.22.0.1
[fedora@fedora32 ~]$ ip route
default via 172.22.0.1 dev eth0 
172.22.0.0/24 dev eth0 proto kernel scope link src 172.22.0.29 metric 100
~~~

Now lets try to ping Google's nameserver:

~~~bash
[fedora@fedora32 ~]$ ping -c 4 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=2.81 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=1.89 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=2.16 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=116 time=1.91 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 1.886/2.192/2.807/0.370 ms
~~~

As you can see we were able to access the outside world!

Feel free to play around with the OpenShift Web Console and the Virtualization menu's - this was a quick play around with all of this, but should time allow, go ahead and see what you can do.

## Success!

Congratulations ... if you've made it this far you've deployed KNI from the ground up, deployed OpenShift Container Storage, OpenShift Virtualization, and tested the solution with pods and VM's via the CLI and the OpenShift dashboard! We'd like to ***thank you*** for attending this lab and hope that it was a valuable use of your time and that you learnt a lot from doing it.

Please do let us know if there's anything else we can do to support you by reaching out to us at [field-engagement@redhat.com](mailto:field-engagement@redhat.com), and as always - pull requests are more than welcome to this repository ;-)

