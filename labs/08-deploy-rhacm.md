# **Deploy Red Hat Advanced Cluster Management (RHACM)**

Red Hat Advanced Cluster Management (RHACM) offers end-to-end visibility and control for managing your cluster and application lifecycle. Among other features, it ensures security and compliance for the entire Kubernetes domain across multiple data centers and public clouds. 

This lab will focus on installing and configuring Red Hat Advanced Cluster Management (RHACM) to enable the AIO OpenShift deployed cluster to provide observability, governement and application management. The mechanism for installation is to utilise the operator model via the cli, just like we did in many of our previous labs. Note, it's entirely possible to deploy via the web UI should you wish to do so, but we're not documenting that mechanism here.

~~~bash
[root@ocp4-bastion ~]# oc create namespace open-cluster-management
namespace/open-cluster-management created
[root@ocp4-bastion ~]# oc project open-cluster-management
Now using project "open-cluster-management" on server "https://api.aio.example.com:6443".
~~~


~~~bash
[root@ocp4-bastion ~]# oc create secret generic pull-secret -n open-cluster-management --from-file=.dockerconfigjson=/root/pull-secret.json --type=kubernetes.io/dockerconfigjson
secret/pull-secret created
~~~

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/acm-operator-group.yaml
 apiVersion: operators.coreos.com/v1
 kind: OperatorGroup
 metadata:
   name: acm-operator
 spec:
   targetNamespaces:
   - open-cluster-management
 EOF
~~~


~~~bash
[root@ocp4-bastion ~]# oc create -f acm-operator-group.yaml
operatorgroup.operators.coreos.com/acm-operator created
~~~

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/acm-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: acm-operator-subscription
spec:
  sourceNamespace: openshift-marketplace
  source: redhat-operators
  channel: release-2.2
  installPlanApproval: Automatic
  name: advanced-cluster-management
EOF
~~~

~~~bash

~~~
