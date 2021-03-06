# **Deploy Red Hat Advanced Cluster Management (RHACM)**

Red Hat Advanced Cluster Management (RHACM) offers end-to-end visibility and control for managing your cluster and application lifecycle. Among other features, it ensures security and compliance for the entire Kubernetes domain across multiple data centers and public clouds. 

This lab will focus on installing and configuring Red Hat Advanced Cluster Management (RHACM) to enable the AIO OpenShift deployed cluster to provide observability, governement and application management. The mechanism for installation is to utilise the operator model via the cli, just like we did in many of our previous labs. Note, it's entirely possible to deploy via the web UI should you wish to do so, but we're not documenting that mechanism here.

> **NOTE**: Due to memory requirements the RHACM lab cannot be used simultaneously with the OCS/CNV labs.  However NFS storage does need to be configured to provide a PV that can be consumed.  See the following for how to configure NFS: [Deploying Additional NFS And Hostpath Storage](https://github.com/RHFieldProductManagement/openshift-aio/blob/main/labs/06-deploy-nfs-hostpath-cnv.md).

To begin the deployment process for RHACM we first need to configure a namespace.  By default when using Operator Hub to install RHACM it will use the open-cluster-management namespace.  Therefore we will go ahead and create that namespace here and then use that project:

~~~bash
[root@ocp4-bastion ~]# oc create namespace open-cluster-management
namespace/open-cluster-management created
[root@ocp4-bastion ~]# oc project open-cluster-management
Now using project "open-cluster-management" on server "https://api.aio.example.com:6443".
~~~

Once we have created the namespace we also need to configure a pull-secret secret that can be consumed by the namespace.  We can do this by creating the following generic secret and point it to our pull-secret.json file that is already on the system when AIO did the initial deployment of the environment:

~~~bash
[root@ocp4-bastion ~]# oc create secret generic pull-secret -n open-cluster-management --from-file=.dockerconfigjson=/root/pull-secret.json --type=kubernetes.io/dockerconfigjson
secret/pull-secret created
~~~

Now we need to proceed by creating an operator group.  To do that we need to construct the following operator group yaml file:

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

Next we can create the operator group with the resource yaml we constructed:

~~~bash
[root@ocp4-bastion ~]# oc create -f acm-operator-group.yaml
operatorgroup.operators.coreos.com/acm-operator created
~~~

Finally we need to build a RHACM operator subscription yaml which instructs which operator to install and at what version from Operator Hub:

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/acm-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: acm-operator-subscription
spec:
  sourceNamespace: openshift-marketplace
  source: redhat-operators
  channel: release-2.3
  installPlanApproval: Automatic
  name: advanced-cluster-management
EOF
~~~

With the subscription file created we can go ahead and apply it to the the OCP cluster:

~~~bash
[root@ocp4-bastion ~]# oc create -f acm-operator-subscription.yaml
subscription.operators.coreos.com/acm-operator-subscription created
~~~

At this point the RHACM operator should be installing onto our AIO deployed OCP cluster.  We can see that by issuing a oc get csv against the open-cluster-management namespace:

~~~bash
[root@ocp4-bastion ~]# oc get csv -n open-cluster-management
NAME                                 DISPLAY                                      VERSION   REPLACES                             PHASE
advanced-cluster-management.v2.2.5   Advanced Cluster Management for Kubernetes   2.2.5     advanced-cluster-management.v2.2.4   Installing
~~~

Further if we look at the pods in the open-cluster-management namespace we can see there are containers in a creating stage:

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

After a few minutes the operator installation should complete.  We can confirm this by looking at the oc get csv again which should be marked as succeeded:

~~~bash
[root@ocp4-bastion ~]# oc get csv -n open-cluster-management
NAME                                 DISPLAY                                      VERSION   REPLACES                             PHASE
advanced-cluster-management.v2.2.5   Advanced Cluster Management for Kubernetes   2.2.5     advanced-cluster-management.v2.2.4   Succeeded
~~~

We can further verify that our pods are running by issuing an oc get pods in the open-cluster-management namespace:

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

Now that the operator is installed its time to actually configure the multicluster hub of the RHACM product.  First lets prepare a multiclusterhub resource yaml with the following content: 

~~~bash
[root@ocp4-bastion ~]# cat << EOF > ~/acm-multiclusterhub.yaml
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
spec:
  imagePullSecret: pull-secret
EOF
~~~

Once we have the resource yaml created we can apply it to the cluster:

~~~bash
[root@ocp4-bastion ~]# oc create -f acm-multiclusterhub.yaml 
multiclusterhub.operator.open-cluster-management.io/multiclusterhub created
~~~

We can view the status of the multiclusterhub configuration by looking at the mch status:

~~~bash
[root@ocp4-bastion ~]# oc get mch -n open-cluster-management
NAME              STATUS       AGE
multiclusterhub   Installing   4s
~~~

And if we look at the pods under the open-cluster-management namespace we can see additional pods around the multiclusterhub are being created:

~~~bash
[root@ocp4-bastion ~]# oc get pods -n open-cluster-management
NAME                                                              READY   STATUS              RESTARTS   AGE
cert-manager-c1259-55d5b947d6-66jkq                               1/1     Running             0          95s
cert-manager-c1259-55d5b947d6-xxzk7                               1/1     Running             0          95s
cert-manager-webhook-79bd7865b8-24mvp                             1/1     Running             0          2m51s
cert-manager-webhook-79bd7865b8-hlmf4                             1/1     Running             0          2m51s
cert-manager-webhook-9ccb4-cainjector-568c669b8f-nmh8c            1/1     Running             0          2m52s
cert-manager-webhook-9ccb4-cainjector-568c669b8f-xclwb            1/1     Running             0          2m51s
cluster-manager-7cfffb878b-5pwgf                                  1/1     Running             0          20h
cluster-manager-7cfffb878b-q96px                                  1/1     Running             0          20h
cluster-manager-7cfffb878b-rw6xm                                  1/1     Running             0          20h
configmap-watcher-7d309-94df7bccb-7nqmz                           0/1     ContainerCreating   0          2s
configmap-watcher-7d309-94df7bccb-9p4pd                           0/1     ContainerCreating   0          2s
hive-operator-dc6d88f8c-44r9p                                     1/1     Running             0          20h
multicluster-observability-operator-85477b6644-z72pq              1/1     Running             0          20h
multicluster-operators-application-744cc4dbb8-nttv9               5/5     Running             7          20h
multicluster-operators-hub-subscription-d98c978cc-dbg2z           1/1     Running             1          20h
multicluster-operators-standalone-subscription-69b755cf4d-z2w4x   1/1     Running             0          20h
multiclusterhub-operator-5b56686f4d-z9vf6                         1/1     Running             0          20h
multiclusterhub-repo-7595475cb6-k4h2r                             1/1     Running             0          3m10s
ocm-controller-786d9df595-5b2zw                                   0/1     ContainerCreating   0          0s
ocm-controller-786d9df595-w8b6p                                   0/1     ContainerCreating   0          1s
ocm-proxyserver-579ff5f45f-829gt                                  0/1     ContainerCreating   0          1s
ocm-proxyserver-579ff5f45f-rscj4                                  0/1     ContainerCreating   0          1s
ocm-webhook-7ffccc5cd6-4xktw                                      0/1     ContainerCreating   0          1s
ocm-webhook-7ffccc5cd6-87hmn                                      0/1     ContainerCreating   0          1s
submariner-addon-6d96c55d7c-fqk9d                                 1/1     Running             0          20h
~~~

After about 5 minutes we should eventually see all the pods running.   If there are pods in a Pending status please invenstigate those to determine why.  In most cases it is because there is not enough memory on the master nodes being used in the AIO environment.  This can be remediated by increasing the memory of each master node in the lab and rebooting the masters one by one.

~~~bash
[root@ocp4-bastion ~]# oc get pods -n open-cluster-management
NAME                                                              READY   STATUS    RESTARTS   AGE
application-chart-9351a-applicationui-844f8b7799-bmbpr            1/1     Running   0          2m46s
application-chart-9351a-applicationui-844f8b7799-dbhwh            1/1     Running   0          2m30s
cert-manager-ce05a-fbd8b68d4-8blvg                                1/1     Running   0          5m45s
cert-manager-ce05a-fbd8b68d4-dv42q                                1/1     Running   0          5m45s
cert-manager-webhook-7bc45ff4b-7txs6                              1/1     Running   0          7m1s
cert-manager-webhook-7bc45ff4b-h9lgn                              1/1     Running   0          7m1s
cert-manager-webhook-e2ca8-cainjector-6649d546d7-r25r7            1/1     Running   0          7m1s
cert-manager-webhook-e2ca8-cainjector-6649d546d7-xh87r            1/1     Running   0          7m1s
cluster-manager-7cfffb878b-5pwgf                                  1/1     Running   0          20h
cluster-manager-7cfffb878b-q96px                                  1/1     Running   0          20h
cluster-manager-7cfffb878b-rw6xm                                  1/1     Running   0          20h
clusterlifecycle-state-metrics-67fb8f8f8c-m7m88                   1/1     Running   0          3m45s
configmap-watcher-ee7aa-5b989bff9b-8c9hp                          1/1     Running   0          4m11s
configmap-watcher-ee7aa-5b989bff9b-npkx5                          1/1     Running   0          4m11s
console-chart-a9b0d-console-v2-7995c84d48-jd7r5                   1/1     Running   0          3m56s
console-chart-a9b0d-console-v2-7995c84d48-r8bh6                   1/1     Running   0          3m56s
console-chart-a9b0d-consoleapi-55b99768c4-t5vvn                   1/1     Running   0          3m56s
console-chart-a9b0d-consoleapi-55b99768c4-vlh86                   1/1     Running   0          3m57s
console-header-d56dbd96b-6652w                                    1/1     Running   0          3m56s
console-header-d56dbd96b-vqpp8                                    1/1     Running   0          3m56s
grc-15593-grcui-5b45ccb8f-swccf                                   1/1     Running   0          3m53s
grc-15593-grcui-5b45ccb8f-zhm9x                                   1/1     Running   0          3m53s
grc-15593-grcuiapi-7d876bc9cf-dfncf                               1/1     Running   0          3m53s
grc-15593-grcuiapi-7d876bc9cf-hbwkf                               1/1     Running   0          3m54s
grc-15593-policy-propagator-76467f9f87-jgnw2                      1/1     Running   0          3m53s
grc-15593-policy-propagator-76467f9f87-mwsbx                      1/1     Running   0          3m54s
hive-operator-dc6d88f8c-44r9p                                     1/1     Running   0          20h
klusterlet-addon-controller-7694c8dd65-pnjdf                      1/1     Running   0          3m45s
klusterlet-addon-controller-7694c8dd65-q6xrf                      1/1     Running   0          3m44s
kui-web-terminal-74b44c69f7-jszx4                                 1/1     Running   0          2m48s
managedcluster-import-controller-76b4987496-9zjwl                 1/1     Running   0          3m44s
managedcluster-import-controller-76b4987496-txrwb                 1/1     Running   0          3m45s
management-ingress-6dd43-5f76bc695-8crnc                          2/2     Running   0          2m48s
management-ingress-6dd43-5f76bc695-r525g                          2/2     Running   0          2m48s
multicluster-observability-operator-85477b6644-z72pq              1/1     Running   0          20h
multicluster-operators-application-744cc4dbb8-nttv9               5/5     Running   9          20h
multicluster-operators-hub-subscription-d98c978cc-dbg2z           1/1     Running   1          20h
multicluster-operators-standalone-subscription-69b755cf4d-z2w4x   1/1     Running   0          20h
multiclusterhub-operator-5b56686f4d-z9vf6                         1/1     Running   0          20h
multiclusterhub-repo-7595475cb6-h4wf5                             1/1     Running   0          7m19s
ocm-controller-786d9df595-2ltx9                                   1/1     Running   0          4m6s
ocm-controller-786d9df595-xdwc8                                   1/1     Running   0          4m6s
ocm-proxyserver-579ff5f45f-76w67                                  1/1     Running   0          4m7s
ocm-proxyserver-579ff5f45f-gd79j                                  1/1     Running   0          4m6s
ocm-webhook-7ffccc5cd6-fkn9h                                      1/1     Running   0          4m8s
ocm-webhook-7ffccc5cd6-sx49s                                      1/1     Running   0          4m7s
search-operator-b7f98dfdb-dgxls                                   1/1     Running   0          3m38s
search-prod-d0d24-search-aggregator-5ff4456fdd-bt25m              1/1     Running   0          2m48s
search-prod-d0d24-search-api-767d966648-24znz                     1/1     Running   0          2m48s
search-prod-d0d24-search-api-767d966648-6d4pq                     1/1     Running   0          94s
search-prod-d0d24-search-collector-57fb7f8459-98wzr               1/1     Running   0          2m48s
search-redisgraph-0                                               1/1     Running   0          119s
search-ui-55d4d5764-br52w                                         1/1     Running   0          3m38s
search-ui-55d4d5764-rt65b                                         1/1     Running   0          3m38s
submariner-addon-6d96c55d7c-fqk9d                                 1/1     Running   0          20h
topology-f68f8-topology-578f846595-9z76h                          1/1     Running   0          2m33s
topology-f68f8-topology-578f846595-qvbq2                          1/1     Running   0          2m33s
topology-f68f8-topologyapi-6ddb677b6f-fpdw4                       1/1     Running   0          2m33s
topology-f68f8-topologyapi-6ddb677b6f-w7vff                       1/1     Running   0          2m33s
~~~

We can also confirm the installation is completed by looking for the routes in the open-cluster-management namespace:

~~~bash
[root@ocp4-bastion ~]# oc get routes -n open-cluster-management
NAME                 HOST/PORT                                 PATH   SERVICES             PORT    TERMINATION            WILDCARD
multicloud-console   multicloud-console.apps.aio.example.com          management-ingress   <all>   passthrough/Redirect   None
~~~

If Guacomole was installed in the AIO environment we can take the multicloud-console.apps.aio.example.com url and paste it into a browser via the Guacomole UI to get the RHACM Console.
