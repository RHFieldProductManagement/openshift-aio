In this section we're going to be configuring the networking for our environment. You're probably just wanting to create virtual machines, but let's just finish this one step and we'll understand a lot more about how networking works in OpenShift.

With OpenShift Virtualization (or more specifically, OpenShift in general - regardless of the workload type) we have a few different options for networking - we can just have our virtual machines be attached to the same pod networks that our containers would have access to, or we can configure more real-world virtualisation networking constructs like bridged networking, SR/IOV, and so on. It's also absolutely possible to have a combination of these, e.g. both pod networking and a bridged interface directly attached to a VM at the same time, using [Multus](https://github.com/k8snetworkplumbingwg/multus-cni), the default networking CNI in OpenShift 4.x. Multus allows multiple "sub-CNI" devices to be attached to a pod (regardless of whether a virtual machine is running there).

In this lab we're going to enable multiple options - pod networking **and** a secondary network interface provided by a bridge on the underlying worker nodes (hypervisors). Each of the worker nodes has been configured with an additional, currently unused, network interface that is defined as `enp3s0`, and we'll create a bridge device, called `br1`, so we can attach our virtual machines to it - this network is actually the same L2 network as the one attached to `enp2s0`, so it's on the `192.168.123.0/24` network as well.

The first step is to use the new Kubernetes NetworkManager state configuration to setup the underlying hosts to our liking. Recall that we can get the **current** state by requesting the `NetworkNodeState` (much of the following is snipped for brevity):


```execute-1
oc get nns/ocp4-worker1.cnv.example.com -o yaml
```

This will display the NodeNetworkState in yaml format

~~~yaml
apiVersion: nmstate.io/v1beta1
kind: NodeNetworkState
metadata:
  creationTimestamp: "2021-11-08T12:11:09Z"
  generation: 1
  labels:
    app.kubernetes.io/component: network
    app.kubernetes.io/managed-by: hco-operator
    app.kubernetes.io/part-of: hyperconverged-cluster
    app.kubernetes.io/version: v4.9.0
  name: ocp4-worker1.%node-network-domain%
(...)
    - ipv4:
        address:
        - ip: 192.168.123.104
          prefix-length: 24
        - ip: 192.168.123.11
          prefix-length: 32
        auto-dns: true
        auto-gateway: true
        auto-route-table-id: 0
        auto-routes: true
        dhcp: true
        enabled: true
      ipv6:
        address:
        - ip: fe80::5054:ff:fe00:4
          prefix-length: 64
        auto-dns: true
        auto-gateway: true
        auto-route-table-id: 0
        auto-routes: true
        autoconf: true
        dhcp: true
        enabled: true
      lldp:
        enabled: false
      mac-address: "52:54:00:00:00:04"
      mtu: 1500
      name: enp2s0
      state: up
      type: ethernet
    - ipv4:
        address: []
        dhcp: false
        enabled: false
      ipv6:
        address:
        - ip: fe80::5054:ff:fe00:104
          prefix-length: 64
        autoconf: false
        dhcp: false
        enabled: true
      lldp:
        enabled: false
      mac-address: "52:54:00:00:01:04"
      mtu: 1500
      name: enp3s0
      state: up
      type: ethernet
(...)
~~~

In there you'll spot the interface that we'd like to use to create a bridge, `enp3s0`, with DHCP being disabled and not in current use - there are no IP addresses associated to that network. DHCP is configured in the environment, but as part of the installation we forced this network to be disabled via [MachineConfig](https://github.com/RHFieldProductManagement/openshift-aio/blob/main/conf/k8s/97_workers_empty_enp3s0.yaml). We'll make the necessary changes to this interface shortly.

> **NOTE**: The first interface, `enp1s0 ` is used for provisioning within the environment, and `enp2s0` is being used for inter-OpenShift communication, including all of the pod networking via OpenShift SDN.


Now we can apply a new `NodeNetworkConfigurationPolicy` for our worker nodes to setup a desired state for `br1` via `enp3s0`, noting that in the `spec` we specify a `nodeSelector` to ensure that this **only** gets applied to our worker nodes; eventually allowing us to attach VM's to this bridge:

```execute-1
$ cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-enp3s0-policy-workers
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with enp3s0 as a port
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: enp3s0
EOF
```

Check the output

~~~bash
nodenetworkconfigurationpolicy.nmstate.io/br1-enp3s0-policy-workers created
~~~

Then enquire as to whether it was successfully applied:

```execute-1
$ oc get nnce
```

Check the status:

~~~bash
NAME                                                     STATUS
ocp4-worker1.%node-network-domain%.br1-enp3s0-policy-workers   Available
ocp4-worker2.%node-network-domain%.br1-enp3s0-policy-workers   Available
ocp4-worker3.%node-network-domain%.br1-enp3s0-policy-workers   Available
~~~

```execute-1
oc get nncp
```

Check the result:
~~~bash
NAME                        STATUS
br1-enp3s0-policy-workers   Available
~~~

We can also dive into the `NetworkNodeConfigurationPolicy` (**nncp**) a little further:

~~~bash
$ oc get nncp/br1-enp3s0-policy-workers -o yaml
apiVersion: nmstate.io/v1beta1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"nmstate.io/v1alpha1","kind":"NodeNetworkConfigurationPolicy","metadata":{"annotations":{},"name":"br1-enp3s0-policy-workers"},"spec":{"desiredState":{"interfaces":[{"bridge":{"options":{"stp":{"enabled":false}},"port":[{"name":"enp3s0"}]},"description":"Linux bridge with enp3s0 as a port","ipv4":{"enabled":false},"name":"br1","state":"up","type":"linux-bridge"}]},"nodeSelector":{"node-role.kubernetes.io/worker":""}}}
    nmstate.io/webhook-mutating-timestamp: "1636377787953660263"
  creationTimestamp: "2021-11-08T13:23:08Z"
  generation: 1
  name: br1-enp3s0-policy-workers
  resourceVersion: "133303"
  uid: 893f9f6e-c447-44b8-821d-73217341c6d6
spec:
  desiredState:
    interfaces:
    - bridge:
        options:
          stp:
            enabled: false
        port:
        - name: enp3s0
      description: Linux bridge with enp3s0 as a port
      ipv4:
        enabled: false
      name: br1
      state: up
      type: linux-bridge
  nodeSelector:
    node-role.kubernetes.io/worker: ""
status:
  conditions:
  - lastHearbeatTime: "2021-11-08T13:23:34Z"
    lastTransitionTime: "2021-11-08T13:23:34Z"
    message: 3/3 nodes successfully configured
    reason: SuccessfullyConfigured
    status: "True"
    type: Available
  - lastHearbeatTime: "2021-11-08T13:23:34Z"
    lastTransitionTime: "2021-11-08T13:23:34Z"
    reason: SuccessfullyConfigured
    status: "False"
    type: Degraded
~~~

If you'd like to fully verify that this has been successfully configured on the host, we can do that easily via the `oc debug node` option (pick any of your workers):

~~~bash
$ oc debug node/ocp4-worker1.%node-network-domain%
Starting pod/ocp4-worker1aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.104
If you don't see a command prompt, try pressing enter.

sh-4.4# ip link show dev enp3s0
4: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br1 state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:00:01:04 brd ff:ff:ff:ff:ff:ff

sh-4.4# exit

Removing debug pod ...

$ oc whoami
system:serviceaccount:workbook:cnv
~~~

As you can see, *enp3s0* is attached to "*br1*" and has the MAC address of the underlying "physical" adaptor.

> **NOTE**: Make sure you exit out of the debug pod before proceeding.

Now that the "physical" networking is configured on the underlying worker nodes, we need to then define a `NetworkAttachmentDefinition` so that when we want to use this bridge, OpenShift and OpenShift Virtualization know how to attach into it. This associates the bridge we just defined with a logical name, known here as '**tuning-bridge-fixed**':

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: tuning-bridge-fixed
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br1
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "groot",
    "plugins": [
      {
        "type": "cnv-bridge",
        "bridge": "br1"
      },
      {
        "type": "tuning"
      }
    ]
  }'
EOF

networkattachmentdefinition.k8s.cni.cncf.io/tuning-bridge-fixed created
~~~

> **NOTE**: The important flags to recognise here are the **type**, being **cnv-bridge** which is a specific implementation that links in-VM interfaces to a counterpart on the underlying host for full-passthrough of networking. Also note that there is no **ipam** listed - we don't want the CNI to manage the network address allocation for us - the network we want to attach to has DHCP enabled, and so let's not get involved.

That's it! We're good to go, next step is to deploy a virtual machine!