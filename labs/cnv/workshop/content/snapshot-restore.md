# Background: Virtual Machine snapshots

A snapshot represents the state and data of a virtual machine (VM) at a specific point in time. You can use a snapshot to restore an existing VM to a previous state (represented by the snapshot) for backup and disaster recovery or to rapidly roll back to a previous development version.

You can create and delete virtual machine (VM) snapshots for VMs, whether the VMs are powered off (**offline**) or on (**online**).

When taking a snapshot of a running VM, the controller checks that the **QEMU guest agent** is installed and running. If so, it freezes the VM file system before taking the snapshot, and thaws the file system after the snapshot is taken.

The snapshot stores a copy of each Container Storage Interface (CSI) volume attached to the VM and a copy of the VM specification and metadata. Snapshots cannot be changed after creation.

With the VM snapshots feature, cluster administrators and application developers can:
- Create a new snapshot
- List all snapshots attached to a specific VM
- Restore a VM from a snapshot
- Delete an existing VM snapshot

OpenShift Virtualization supports VM snapshots on the following:
- Red Hat OpenShift Container Storage
- Any other storage provider with the Container Storage Interface (CSI) driver that supports the Kubernetes Volume Snapshot API

> **NOTE**: We're calling most of the resources "RHEL 8" here, regardless of whether you're actually using CentOS 8 as your base image - it won't impact anything for our purposes here.

# Exercise: Installing QEMU guest agent

To create snapshots of an online (Running state) VM with the highest integrity, install the QEMU guest agent.

The QEMU guest agent takes a consistent snapshot by attempting to quiesce the VM’s file system as much as possible, depending on the system workload. This ensures that in-flight I/O is written to the disk before the snapshot is taken. If the guest agent is not present, quiescing is not possible and a best-effort snapshot is taken. The conditions under which the snapshot was taken are reflected in the snapshot indications that are displayed in the web console or CLI.

> **NOTE**: The qemu-guest-agent is widely available and available by default in Red Hat virtual machines. It might be already installed and enabled on the virtual machine used in this lab module.

1. Navigate to the OpenShift Web UI so we can access the console of the `mongodb-nationalparks` virtual machine. You'll need to select "**Workloads**" --> "**Virtualization**" --> "**Virtual Machines**" --> "**mongodb-nationalparks**" --> "**Console**". You'll be able to login with "**centos/redhat**", noting that you may have to click on the console window for it to capture your input. <img src="img/snapshot-vm-console.png"/>

> **TIP**: You might find `Serial Console` option is more responsive.

> **NOTE**: If you don't see an VMs make sure to change to the Default project via the drop down at the top of the console.

2. Once you're in the virtual machine Install the QEMU guest agent on the virtual machine
~~~bash
sudo yum install -y qemu-guest-agent
~~~

3. Ensure the service is persistent and start it
~~~bash
sudo systemctl enable --now qemu-guest-agent
~~~

# Exercise: Creating a virtual machine snapshot in the web console

Virtual machine (VM) snapshots can be created either by using the web console or in the CLI. In this exercise, let's create a snapshot of our mongodb database vm by using the web console.
<table>
  <tr>
    <td>
<img src="img/vm-snapshot-card.png" width="40%" align="right"/>

1. Click **Workloads** → **Virtualization** from the side menu.
   
2. Click the **Virtual Machines** tab.
   
3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. Click the **Snapshots** tab and then click **Take Snapshot**.

5. Fill in the **Snapshot Name** and optional **Description** fields.

6. Because the VM has a cloud-init disk that cannot be included in the snapshot, select the **I am aware of this warning and wish to proceed** checkbox.

7. Click **Save**.
   </td>
  </tr>
 </table>

Once you click Save to create snapshot, the vm controller checks that the QEMU guest agent is installed and running. If so, it freezes the VM file system before taking the snapshot, and initiates snapshot creation on actual storage system for each Container Storage Interface (CSI) volume attached to the VM, a copy of the VM specification and metadata is also created.

<img src="img/vm-snapshot-creating.png"/><br><br> 

It should take just a little seconds to actually create the snapshot and make it Ready to use. Once the snapshot becomes **Ready** then it can be used to restore the virtual machine to that specific point in time then the snapshot is taken.

<img src="img/vm-snapshot-ready.png"/>

# Exercise: Destroy database

After taking an online snapshot of the database vm, let's destroy the database by forcefully deleting everything under it's data path.

1. Navigate to the OpenShift Web UI so we can access the console of the `mongodb-nationalparks` virtual machine. You'll need to select "**Workloads**" --> "**Virtualization**" --> "**Virtual Machines**" --> "**mongodb-nationalparks**" --> "**Console**". You'll be able to login with "**centos/redhat**", noting that you may have to click on the console window for it to capture your input.

2. Once you're in the virtual machine, delete everything under it's data path.
~~~bash
sudo systemctl stop mongod
~~~
~~~bash
sudo rm -rf /var/lib/mongo/*
~~~
~~~bash
sudo systemctl start mongod
~~~

Now you can check by refreshing `ParksMap` web page (Map Visualizer on OpenShift 4), it should **fail** to load national parks locations from the backend service and no longer display them on the map.

# Exercise: Restoring a virtual machine from a snapshot in the web console

In this exercise, let's restore our mongodb database vm by using the web console to the snapshot created in the previous exercise.
You can only restore to a powered off (offline) VM so we will first power off the virtual machine in this exercise.

1. Click **Workloads** → **Virtualization** from the side menu.

2. Click the **Virtual Machines** tab.

3. Select `mongodb-nationalparks` virtual machine to open its **Overview** screen.

4. If the virtual machine is running, click **Actions** → **Stop Virtual Machine** to power it down.

5. Click the **Snapshots** tab. The page displays a list of snapshots associated with the virtual machine.

6. Choose one of the following methods to restore a VM snapshot:

   - For the snapshot that you want to use as the source to restore the VM, click **Restore**.
   - Select a snapshot to open the **Snapshot Details** screen and click **Actions** → **Restore Virtual Machine Snapshot**.

7. In the confirmation pop-up window, click **Restore** to restore the VM to its previous configuration represented by the snapshot.

<img src="img/vm-snapshot-restore-popup.png"  width="70%"/><br>

Once you click Restore to restore vm from the snapshot, it initiates snapshot restoration on actual storage system for each Container Storage Interface (CSI) volume attached to the VM and included in the snaphot, VM specification and metadata is also restored.  
It should take just a little seconds to actually restore the snapshot and make the VM ready to be powered on again.

<img src="img/vm-snapshot-restoring.png"/><br>

After the snapshot was restored successfully and it's status become **Ready**, then you can click **Actions** → **Start Virtual Machine** to power it on.
Once the VM is powered on and boots successfully, you can refresh `ParksMap` web page (Map Visualizer on OpenShift 4). It should successfully load national parks locations from the backend service and start displaying them on the map again.

# Background: Virtual machine snapshot controller and custom resource definitions (CRDs)

The VM snapshot feature introduces three new API objects defined as CRDs for managing snapshots:

- `VirtualMachineSnapshot`: Represents a user request to create a snapshot. It contains information about the current state of the VM.

- `VirtualMachineSnapshotContent`: Represents a provisioned resource on the cluster (a snapshot). It is created by the VM snapshot controller and contains references to all resources required to restore the VM.

- `VirtualMachineRestore`: Represents a user request to restore a VM from a snapshot.

The VM snapshot controller binds a `VirtualMachineSnapshotContent` object with the `VirtualMachineSnapshot` object for which it was created, with a one-to-one mapping.

# Exercise: Creating an virtual machine snapshot in the CLI

In previous exercises in this module, we created and restored vm snapshot in the OpenShift web console. However, It's also possible to do same operations in the CLI using the CRDs above. Using CLI and Yaml/Json definitions of `VirtualMachineSnapshot` and `VirtualMachineRestore` objects to create and restore snapshot respectively, allows automating all snapshot releated operations.

In this exercise, let's create another snapshot of our mongodb database vm, this time by using the cli.

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: snapshot.kubevirt.io/v1alpha1
kind: VirtualMachineSnapshot
metadata:
  name: my-vmsnapshot 
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: my-vm 
EOF
~~~

That's it for deploying basic workloads - we've deployed a VM on-top of OCS and one on-top of hostpath.
=======