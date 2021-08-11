# **Running Workloads in the Environment**

Let's get into actually testing some workloads on our environment... we've spent all of this time building it up but we haven't even proven that it's working properly yet! In this section we're going to be deploying some pods as well as deploying a VM using OpenShift Virtualization.

## Deploying a Virtual Machine Using OCS and Bridge Net

In the previous deploy OpenShift Virtualization lab we installed the OpenShift Virtualization operator and created a virtualization cluster.  We also configured an external bridge so that our virtual machines can be connected to the outside network.  

We are now at the point where we can launch a virtual machine!  To do this we will use the following YAML file below, recalling that as with all things OpenShift, we can deploy virtual machines in the same way as we would standard containerised pods.  Lets go ahead and create the file:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/virtualmachine-fedora.yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/fedora33: Fedora 32 or higher
    vm.kubevirt.io/flavor: small
    vm.kubevirt.io/os: fedora
    vm.kubevirt.io/validations: |
      [
        {
          "name": "minimal-required-memory",
          "path": "jsonpath::.spec.domain.resources.requests.memory",
          "rule": "integer",
          "message": "This VM requires more memory.",
          "min": 1073741824
        }
      ]
    vm.kubevirt.io/workload: server
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachines/fedora34
  resourceVersion: '818848'
  name: fedora34
  uid: 781fc602-eb87-459e-addc-47d1b7b4f9da
  creationTimestamp: '2021-08-11T18:00:14Z'
  generation: 1
  managedFields:
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            .: {}
            'f:name.os.template.kubevirt.io/fedora33': {}
            'f:vm.kubevirt.io/flavor': {}
            'f:vm.kubevirt.io/os': {}
            'f:vm.kubevirt.io/validations': {}
            'f:vm.kubevirt.io/workload': {}
          'f:labels':
            'f:vm.kubevirt.io/template.version': {}
            'f:vm.kubevirt.io/template.namespace': {}
            'f:app': {}
            .: {}
            'f:vm.kubevirt.io/template.revision': {}
            'f:workload.template.kubevirt.io/server': {}
            'f:flavor.template.kubevirt.io/small': {}
            'f:os.template.kubevirt.io/fedora33': {}
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
                'f:flavor.template.kubevirt.io/small': {}
                'f:kubevirt.io/domain': {}
                'f:kubevirt.io/size': {}
                'f:os.template.kubevirt.io/fedora33': {}
                'f:vm.kubevirt.io/name': {}
                'f:workload.template.kubevirt.io/server': {}
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
      time: '2021-08-11T18:00:14Z'
    - apiVersion: kubevirt.io/v1alpha3
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:annotations':
            'f:kubevirt.io/latest-observed-api-version': {}
            'f:kubevirt.io/storage-observed-api-version': {}
        'f:status':
          .: {}
          'f:volumeSnapshotStatuses': {}
      manager: virt-controller
      operation: Update
      time: '2021-08-11T18:00:15Z'
  namespace: default
  labels:
    app: fedora34
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/fedora33: 'true'
    vm.kubevirt.io/template: fedora-server-small
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.13.2
    workload.template.kubevirt.io/server: 'true'
spec:
  dataVolumeTemplates:
    - metadata:
        creationTimestamp: null
        name: fedora34-rootdisk-nsh5m
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
              https://download-ib01.fedoraproject.org/pub/fedora/linux/releases/34/Cloud/x86_64/images/Fedora-Cloud-Base-34-1.2.x86_64.raw.xz
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/small: 'true'
        kubevirt.io/domain: fedora34
        kubevirt.io/size: small
        os.template.kubevirt.io/fedora33: 'true'
        vm.kubevirt.io/name: fedora34
        workload.template.kubevirt.io/server: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          autoattachPodInterface: false
          disks:
            - disk:
                bus: virtio
              name: cloudinitdisk
            - bootOrder: 1
              disk:
                bus: virtio
              name: rootdisk
          interfaces:
            - bridge: {}
              model: virtio
              name: default
          networkInterfaceMultiqueue: true
          rng: {}
        machine:
          type: pc-q35-rhel8.3.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: fedora34
      networks:
        - multus:
            networkName: brext
          name: default
      terminationGracePeriodSeconds: 180
      volumes:
        - cloudInitNoCloud:
            userData: |
              #cloud-config
              user: fedora
              password: 5sxb-q8jw-lrhk
              chpasswd:
                expire: false
          name: cloudinitdisk
        - dataVolume:
            name: fedora34-rootdisk-nsh5m
          name: rootdisk
status:
  volumeSnapshotStatuses:
    - enabled: false
      name: cloudinitdisk
      reason: Volume type does not suport snapshots
    - enabled: true
      name: rootdisk
EOF
~~~

Let's take a look at some important parts of this file!

We can tell OpenShift Virtualization to use some preset defaults, much like with Red Hat Virtualization. In the below section, among other things, we set a flavor of "small" which provides some basic. pre-defined resource settings for things like CPU and memory, as well as the "workload" type to "server". This also prevides preset configuration to define resources suitable for a desktop machine as opposed to a server.

~~~bash
  labels:
    app: fedora34
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/fedora33: 'true'
    vm.kubevirt.io/template: fedora-server-small
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.13.2
    workload.template.kubevirt.io/server: 'true'
~~~

In this next section we are utilizing the storage environment we built in previous labs! In this case we are creating the root disk of our VM to be created as a PVC using our OpenShift Container Storage install. We do this by setting the **storageClassName** to **ocs-storagecluster-ceph-rbd** allowing us to access the Ceph install. We can even set the access mode to ReadWriteMany meaning this VM can be live migrated easily. Finally, we source URL to tell OpenShift Virtualization where to get an OS image to load into the PVC:


~~~bash
  dataVolumeTemplates:
    - metadata:exit
        creationTimestamp: null
        name: fedora34-rootdisk-nsh5m
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
              https://download-ib01.fedoraproject.org/pub/fedora/linux/releases/34/Cloud/x86_64/images/Fedora-Cloud-Base-34-1.2.x86_64.raw.xz
~~~

The storage info about works with the snippet below. In that, we create a dataVolume utilising the PVC containing the pre-loaded Operating System we download. When we boot the VM it will use that OS!

~~~bash
        - dataVolume:
            name: fedora34-rootdisk-nsh5m
          name: rootdisk
~~~

Another neat thing we can do with OpenShift Virtualization is use cloud-config. Cloud-config allows us to set important funtionality into a metadata service that is read and actioned when the VM is first booted. In the case below, we are setting the new VMs hostname to "**fedora34**" and it's root password to "**redhat**".

~~~bash
        - cloudInitNoCloud:
            userData: |
              #cloud-config
              user: fedora
              password: 5sxb-q8jw-lrhk
              chpasswd:
                expire: false
          name: cloudinitdisk
~~~

Finally let's call out two snippets that set up our networking. In this first section we define the first (default) interface on the machine:


~~~bash
          interfaces:
            - bridge: {}
              model: virtio
              name: default
~~~

We are then able to use the the following section to instruct OpenShift to utilise its default Container Network Interface (CNI) network provider, **multus**, to connect that NIC into the **brext** bridge we created in the previous labs. This ensure the VM is exposed to the external network and not just the internal OpenStack SDN (like pods are).


~~~bash
      networks:
        - multus:
            networkName: brext
          name: default
~~~

Now we can go ahead and create the virtual machine:

~~~bash
[root@ocp4-bastion ~]# oc create -f virtualmachine-fedora.yaml 
virtualmachine.kubevirt.io/fedora34 created
~~~

If we run a oc get pods for the default namespace we will see an importer container.  Once it is running we can watch the logs:

~~~bash
[root@ocp4-bastion ~]# oc get pods
NAME                               READY   STATUS    RESTARTS   AGE
importer-fedora34-rootdisk-nsh5m   1/1     Running   0          5m5s
~~~

You can following the image importing process container which is pulling in the fedora image from the URL we had inside the *virtualmachine-fedora.yaml* file:

~~~bash
[root@ocp4-bastion ~]# oc logs importer-fedora34-rootdisk-nsh5m -f
I0811 18:18:22.092726       1 importer.go:52] Starting importer
I0811 18:18:22.094714       1 importer.go:134] begin import process
I0811 18:18:22.419839       1 data-processor.go:324] Calculating available size
I0811 18:18:22.421427       1 data-processor.go:332] Checking out block volume size.
I0811 18:18:22.421449       1 data-processor.go:344] Request image size not empty.
I0811 18:18:22.421506       1 data-processor.go:349] Target size 10Gi.
I0811 18:18:22.475709       1 data-processor.go:232] New phase: TransferDataFile
I0811 18:18:22.478275       1 util.go:161] Writing data...
I0811 18:18:23.476105       1 prometheus.go:69] 0.23
I0811 18:18:24.477360       1 prometheus.go:69] 0.36
I0811 18:18:25.478382       1 prometheus.go:69] 0.36
I0811 18:18:26.479314       1 prometheus.go:69] 0.36
I0811 18:18:27.479700       1 prometheus.go:69] 0.37
I0811 18:18:28.479975       1 prometheus.go:69] 0.37
I0811 18:18:29.480743       1 prometheus.go:69] 0.81
I0811 18:18:30.481615       1 prometheus.go:69] 1.92
I0811 18:18:31.482380       1 prometheus.go:69] 2.13
I0811 18:18:32.483498       1 prometheus.go:69] 2.13
I0811 18:18:33.484408       1 prometheus.go:69] 2.13
I0811 18:18:34.484757       1 prometheus.go:69] 2.13
I0811 18:18:35.485005       1 prometheus.go:69] 2.14
I0811 18:18:36.485197       1 prometheus.go:69] 2.14
I0811 18:18:37.486457       1 prometheus.go:69] 2.14
I0811 18:18:38.486755       1 prometheus.go:69] 5.18
(...)
I0811 18:23:35.702430       1 prometheus.go:69] 99.98
I0811 18:23:36.703387       1 prometheus.go:69] 99.98
I0811 18:23:37.703694       1 prometheus.go:69] 99.98
I0811 18:23:38.704160       1 prometheus.go:69] 99.98
I0811 18:23:39.704418       1 prometheus.go:69] 99.99
I0811 18:23:40.706621       1 prometheus.go:69] 99.99
I0811 18:23:41.707624       1 prometheus.go:69] 99.99
I0811 18:23:42.708701       1 prometheus.go:69] 99.99
I0811 18:23:43.709239       1 prometheus.go:69] 99.99
I0811 18:23:44.709589       1 prometheus.go:69] 99.99
I0811 18:23:45.709854       1 prometheus.go:69] 100.00
I0811 18:23:46.710443       1 prometheus.go:69] 100.00
I0811 18:23:47.710666       1 prometheus.go:69] 100.00
I0811 18:23:51.302690       1 data-processor.go:232] New phase: Resize
I0811 18:23:51.307314       1 data-processor.go:232] New phase: Complete
I0811 18:23:51.307482       1 importer.go:212] Import Complete

~~~

Once the importer process has completed the virtual machine will then begin to startup and go to a running state. To interact with the VM from the CLI we can install a special tool provied with OpenShift Virtualiation called `virtctl`. This CLI is like `oc` or `kubectl` but for working with VMs. Let's install virtctl before we proceed:

~~~bash
[root@ocp4-bastion ~]# rpm -ivh kubevirt-virtctl-4.8.0-226.el8.x86_64.rpm 
warning: kubevirt-virtctl-4.8.0-226.el8.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID fd431d51: NOKEY
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:kubevirt-virtctl-4.8.0-226.el8   ################################# [100%]
~~~

Now use `virtctl` to interact with the virtual machine:

~~~bash
[root@ocp4-bastion ~]# virtctl -h
virtctl controls virtual machine related operations on your kubernetes cluster.

Available Commands:
  addvolume    add a volume to a running VM
  console      Connect to a console of a virtual machine instance.
  expose       Expose a virtual machine instance, virtual machine, or virtual machine instance replica set as a new service.
  fslist       Return full list of filesystems available on the guest machine.
  guestosinfo  Return guest agent info about operating system.
  help         Help about any command
  image-upload Upload a VM image to a DataVolume/PersistentVolumeClaim.
  migrate      Migrate a virtual machine.
  pause        Pause a virtual machine
  removevolume remove a volume from a running VM
  restart      Restart a virtual machine.
  start        Start a virtual machine.
  stop         Stop a virtual machine.
  unpause      Unpause a virtual machine
  userlist     Return full list of logged in users on the guest machine.
  version      Print the client and server version information.
  vnc          Open a vnc connection to a virtual machine instance.

Use "virtctl <command> --help" for more information about a given command.
Use "virtctl options" for a list of global command-line options (applies to all commands).
~~~

Let's go ahead and see what the status of our virtual machine by connecting to the console (you may need to hit enter a couple of times):

~~~bash
[root@ocp4-bastion ~]# virtctl console fedora34
Successfully connected to fedora34 console. The escape sequence is ^]

fedora34 login: 
~~~

As defined by the cloud-config information in our virtual machine YAML file we used the password should be set to **5sxb-q8jw-lrhk** for the **fedora** user.  Lets login:

~~~bash
fedora34 login: fedora
Password: 
Last login: Wed Aug 11 18:26:45 on tty1
[fedora@fedora34 ~]$ cat /etc/fedora-release
Fedora release 34 (Thirty Four)
~~~

Now lets see if we got a 192.168.100.0/24 network address.  If you reference back to the virtual machines YAML file you will notice we used the brext bridge interface as the interface our virtual machine should be connected to.  Thus we should have a 192.168.100.0/24 network address and access outside:

~~~bash
[fedora@fedora34 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 76:a6:5a:a3:68:7d brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 192.168.123.64/24 brd 192.168.123.255 scope global dynamic noprefixroute eth0
       valid_lft 42245sec preferred_lft 42245sec
    inet6 fe80::74a6:5aff:fea3:687d/64 scope link 
       valid_lft forever preferred_lft forever

~~~

Lets see if we can ping the gateway of 192.168.123.1:

~~~bash
[fedora@fedora34 ~]$ ping 192.168.123.1 -c 4
PING 192.168.123.1 (192.168.123.1) 56(84) bytes of data.
64 bytes from 192.168.123.1: icmp_seq=1 ttl=64 time=0.656 ms
64 bytes from 192.168.123.1: icmp_seq=2 ttl=64 time=1.18 ms
64 bytes from 192.168.123.1: icmp_seq=3 ttl=64 time=0.378 ms
64 bytes from 192.168.123.1: icmp_seq=4 ttl=64 time=0.466 ms

--- 192.168.123.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 0.378/0.669/1.177/0.309 ms
~~~

Looks like we have successful external network connectivity!

Before we move on though lets change one thing on the Fedora 34 host.  That change will permit fedora to login with a password when we try to access externally.  We will use a simple inline sed replacement and restart sshd:

~~~bash
[fedora@fedora34 ~]$ sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
[fedora@fedora34 ~]$ sudo systemctl restart sshd
[fedora@fedora34 ~]$ 
~~~

Now lets escape (using `ctrl-]`) out of the console session and connect to the fedora instance via ssh from the bastion host:

~~~bash
[fedora@fedora34 ~]$ [root@ocp4-bastion ~]# ssh fedora@192.168.123.64
The authenticity of host '192.168.123.64 (192.168.123.64)' can't be established.
ECDSA key fingerprint is SHA256:R2Z1M87TNKBdnfFp3cKOUalQZe3/o4kwCm9O7WcqSBo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.123.64' (ECDSA) to the list of known hosts.
fedora@192.168.123.64's password: 
Last login: Wed Aug 11 19:07:59 2021 from 192.168.123.100
[fedora@fedora34 ~]$ 
~~~

Now lets try to ping Google's nameserver:

~~~bash
[fedora@fedora34 ~]$ ping 8.8.8.8 -c 4
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=54 time=8.26 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=54 time=8.26 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=54 time=8.13 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=54 time=8.29 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 8.125/8.235/8.294/0.065 ms
[fedora@fedora34 ~]$ 
[fedora@fedora34 ~]$ 
[fedora@fedora34 ~]$ exit
logout
Connection to 192.168.123.64 closed.
~~~

As you can see we were able to access the outside world!

## Success!

Congratulations ... if you've made it this far you've deployed KNI from the ground up, deployed OpenShift Container Storage, OpenShift Virtualization, and tested the solution with pods and VM's via the CLI and the OpenShift dashboard! We'd like to ***thank you*** for attending this lab and hope that it was a valuable use of your time and that you learnt a lot from doing it.

Please do let us know if there's anything else we can do to support you by reaching out to us at [field-engagement@redhat.com](mailto:field-engagement@redhat.com), and as always - pull requests are more than welcome to this repository ;-)

