Up to this point we've provisioned our virtual machines on a single bridged network, i.e. just using more traditional networking models that you may typically encounter in traditional virtualisation environments. OpenShift 4.x utilises Multus as the default CNI, which permits the user to attach multiple network interfaces from different "delegate CNI's" simultaneously. Therefore, one of the models available for OpenShift Virtualization is to provide networking with a combination of attachments, including pod networking, i.e. having virtual machines being attached to the exact same networks that the container pods are attached to. This has the added benefit of allowing virtual machines to leverage all of the Kubernetes models for services, load balancers, ingress, network policies, node ports, and a wide variety of other functions.

Pod networking is also referred to as "masquerade" mode when it's related to OpenShift Virtualization, and it can be used to hide a virtual machine’s outgoing traffic behind the pod IP address. Masquerade mode uses Network Address Translation (NAT) to connect virtual machines to the Pod network backend through a Linux bridge. Masquerade mode is the recommended binding method for VM's that need to use (or have access to) the default pod network.

Utilising pod networking requires the interface to connect using the `masquerade: {}` method and for IPv4 addresses to be allocated via DHCP. We are going to test this with one of the same Fedora images (or PVC's) we used in the previous lab section. In our virtual machine configuration file we need to ensure the following; instruct the machine to use masquerade mode for the interface (there's no command to execute here, just for your information):

~~~
interfaces:
  - name: nic0				      	            
    model: virtio					              
    masquerade: {}
~~~

Connect the interface to the pod network:

~~~bash
networks:
  - name: nic0
    pod: {}
~~~

So let's go ahead and create a `VirtualMachine` using our existing Fedora 34 image via a PVC we created previously. *Look closely, we are using our cloned PVC so we get the benefits of the installed **NGINX** server, qemu-guest-agent and ssh configuration!*

```execute-1
cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: fc34-podnet
  labels:
    app: fc34-podnet
    os.template.kubevirt.io/fedora34: 'true'
    vm.kubevirt.io/template-namespace: openshift
    workload.template.kubevirt.io/server: 'true'
spec:
  running: true
  template:
    metadata:
      labels:
        vm.kubevirt.io/name: fc34-podnet
        flavor.template.kubevirt.io/small: 'true'
        kubevirt.io/size: small
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: disk0
          interfaces:
            - name: nic0
              model: virtio
              masquerade: {}
          networkInterfaceMultiqueue: true
          rng: {}
        machine:
          type: pc-q35-rhel8.1.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: fc34-podnet
      networks:
        - name: nic0
          pod: {}
      terminationGracePeriodSeconds: 0
      volumes:
        - name: disk0
          persistentVolumeClaim:
            claimName: fc34-original
EOF
```

This should start a new VM:

~~~bash
virtualmachine.kubevirt.io/fc34-podnet created
~~~

We can see the Virtual Machine Instance is created on the pod network, note the IP address in the 10.12x range:

```execute-1
oc get vmi
```

We should then see our VM running:

~~~bash
NAME          AGE   PHASE     IP             NODENAME                       READY
fc34-podnet   68s   Running   10.129.2.210   ocp4-worker2.aio.example.com   True
~~~

If you recall, all VMs are managed by pods, and the pod manages the networking. So we can ask the pod associated with the VM for its IP address, and we'll see that it will match:

```execute
oc get pods -o wide
```

Check the VM's IP:

```execute-1
oc get pods | grep fc31-podnet
```

~~~bash
NAME                              READY   STATUS    RESTARTS   AGE   IP             NODE                           NOMINATED NODE   READINESS GATES
virt-launcher-fc34-podnet-cxztw   1/1     Running   0          20m   10.129.2.210   ocp4-worker2.aio.example.com   <none>           <none>
~~~

We can also check the *pod* for the networks-status, showing the IP address that's being managed, adjust to suit your config:

~~~bash
$ oc describe pod/virt-launcher-fc34-podnet-cxztw | grep -A 9 networks-status
              k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.129.2.210"
                    ],
                    "default": true,
                    "dns": {}
                }]
~~~

As this lab guide is being hosted within the same cluster, you should be able to ping and connect into this VM directly from the terminal window on this IP, adjust to suit your config:


```copy
ping -c1 10.129.2.210
```

~~~bash
$ ping -c1 10.129.2.210
PING 10.129.2.210 (10.129.2.210) 56(84) bytes of data.
64 bytes from 10.129.2.210: icmp_seq=1 ttl=63 time=1.69 ms

--- 10.129.2.210 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.692/1.692/1.692/0.000 ms
~~~



~~~bash
$ ssh root@10.129.2.210
root@10.129.2.210's password:
(password is "redhat")

[root@fc34-podnet ~]# ip a s eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:b1:ce:00:00:05 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.2/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 86309257sec preferred_lft 86309257sec
    inet6 fe80::b1:ceff:fe00:5/64 scope link
       valid_lft forever preferred_lft forever

[root@fc34-podnet ~]# exit
logout
Connection to 10.129.2.210 closed.

$ oc whoami
system:serviceaccount:workbook:cnv
~~~

**Wait a second**... look at the IP address that's been assigned to our VM and the one it recognises... it's different to the one we connected on. Well, this is the masquerading in action - our host is masquerading (on all ports) from the pod network to the network given to our VM (**10.0.2.2** in our case).

This becomes even more evident if we curl the IP address of our VM on the pod network, recalling that we installed NGINX on this VM's disk image in an earlier lab step, you'll see that we curl on the pod IP, but it shows the server address as something different:

~~~bash
$ curl http://10.129.2.210
Server address: 10.0.2.2:80
Server name: fc34-podnet
Date: 06/Dec/2021:13:06:04 +0000
URI: /
Request ID: ae6332e46227c84fe604b6f5c9ec0822
~~~

## Exposing the VM to the outside world

In this step we're going to interface our VM to the outside world using OpenShift/Kubernetes networking constructs, namely services and routes; this will make our VM available via the OpenShift ingress service and you should be able to hit our VM from the internet. As validated in the previous step, our VM has NGINX running on port 80, let's use the `virtctl` utility to expose the virtual machine instance on that port. First `expose` it on port 80 and create a service (an entrypoint) based on our VM:

```execute-1
virtctl expose virtualmachineinstance fc34-podnet --name fc34-service --port 80
```

This should create a new service for us:

~~~bash
Service fc34-service successfully exposed for virtualmachineinstance fc34-podnet
~~~

```execute-1
oc get svc/fc34-service
```

You'll see that we have a new service with a cluster IP associated, listening on port 80:

~~~bash
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
fc34-service    ClusterIP   172.30.31.70    <none>        80/TCP    34s
~~~

Next we create a route for our service, this will associate a URL that we can connect to:

```execute-1
oc create route edge --service=fc34-service
```

A new route will be created:

~~~bash
route.route.openshift.io/fc34-service created
~~~

And view the route:

```execute-1
oc get routes
```

Here you can see the output of our route with the hostname generated based on our cluster:

~~~bash
NAME           HOST/PORT                                              PATH   SERVICES       PORT    TERMINATION   WILDCARD
fc34-service   fc34-service-default.apps.%cluster_subdomain%        fc34-service   <all>   edge          None
~~~

You can now visit the endpoint at [https://fc34-service-default.apps.%cluster_subdomain%/](https://fc34-service-default.apps.%cluster_subdomain%/) in a new browser tab and find the NGINX server from your Fedora based VM - you should see the same content that we curl'd in a previous step, just now it's exposed on the internet:

<img src="img/masq-https.png"/>

> **NOTE**: If you get an "Application is not available" message, make sure that you're accessing the route with **https** - the router performs TLS termination for us, and therefore there's not actually anything listening on port 80 on the outside world, it just forwards 443 (OpenShift ingress) -> 80 (pod network).

There we go, we've successfully exposed our VM onto the internet via the pod network, just like any other containerised application. Let's cleanup this VM before proceeding:

```execute-1
oc delete vm/fc34-podnet
```