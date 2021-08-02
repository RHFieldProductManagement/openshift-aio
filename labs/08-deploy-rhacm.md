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
[root@ocp4-bastion ~]# oc create -f acm-operator-subscription.yaml
subscription.operators.coreos.com/acm-operator-subscription created
~~~

~~~bash
[root@ocp4-bastion ~]# oc get csv -n open-cluster-management
NAME                                 DISPLAY                                      VERSION   REPLACES                             PHASE
advanced-cluster-management.v2.2.5   Advanced Cluster Management for Kubernetes   2.2.5     advanced-cluster-management.v2.2.4   Installing
~~~

~~~bash
[root@ocp4-bastion ~]# oc get pods -n open-cluster-management
NAME                                                              READY   STATUS              RESTARTS   AGE
cluster-manager-7cfffb878b-5pwgf                                  0/1     Running             0          22s
cluster-manager-7cfffb878b-q96px                                  1/1     Running             0          22s
cluster-manager-7cfffb878b-rw6xm                                  0/1     Running             0          22s
hive-operator-dc6d88f8c-44r9p                                     0/1     ContainerCreating   0          20s
multicluster-observability-operator-85477b6644-z72pq              0/1     ContainerCreating   0          22s
multicluster-operators-application-744cc4dbb8-nttv9               0/5     ContainerCreating   0          18s
multicluster-operators-hub-subscription-d98c978cc-dbg2z           0/1     ContainerCreating   0          19s
multicluster-operators-standalone-subscription-69b755cf4d-z2w4x   0/1     ContainerCreating   0          19s
multiclusterhub-operator-5b56686f4d-z9vf6                         0/1     ContainerCreating   0          22s
submariner-addon-6d96c55d7c-fqk9d                                 0/1     ContainerCreating   0          21s
~~~

~~~bash
[root@ocp4-bastion ~]# oc get pods -n open-cluster-management
NAME                                                              READY   STATUS    RESTARTS   AGE
cluster-manager-7cfffb878b-5pwgf                                  1/1     Running   0          3m6s
cluster-manager-7cfffb878b-q96px                                  1/1     Running   0          3m6s
cluster-manager-7cfffb878b-rw6xm                                  1/1     Running   0          3m6s
hive-operator-dc6d88f8c-44r9p                                     1/1     Running   0          3m4s
multicluster-observability-operator-85477b6644-z72pq              1/1     Running   0          3m6s
multicluster-operators-application-744cc4dbb8-nttv9               4/5     Running   1          3m2s
multicluster-operators-hub-subscription-d98c978cc-dbg2z           1/1     Running   0          3m3s
multicluster-operators-standalone-subscription-69b755cf4d-z2w4x   1/1     Running   0          3m3s
multiclusterhub-operator-5b56686f4d-z9vf6                         1/1     Running   0          3m6s
submariner-addon-6d96c55d7c-fqk9d                                 1/1     Running   0          3m5s
~~~
