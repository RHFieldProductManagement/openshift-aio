On the right hand side where the web terminal is, let's see if we can execute the following:

```execute
oc get nodes
```

You should be able to see the list of nodes as below:

~~~bash
NAME                           STATUS   ROLES    AGE   VERSION
ocp4-master1.%node-network-domain%  Ready    master   57m   v1.22.0-rc.0+a44d0f0
ocp4-master2.%node-network-domain%  Ready    master   58m   v1.22.0-rc.0+a44d0f0
ocp4-master3.%node-network-domain% Ready    master   58m   v1.22.0-rc.0+a44d0f0
ocp4-worker1.%node-network-domain%  Ready    worker   38m   v1.22.0-rc.0+a44d0f0
ocp4-worker2.%node-network-domain%  Ready    worker   38m   v1.22.0-rc.0+a44d0f0
ocp4-worker3.%node-network-domain%  Ready    worker   38m   v1.22.0-rc.0+a44d0f0
~~~

If you do not see **three** masters and **three** workers listed in your output, you may need to approve the CSR requests, note that you only need to do this if you're missing nodes, but it won't harm to run this regardless:

```execute
for csr in $(oc get csr | awk '/Pending/ {print $1}'); \
    do oc adm certificate approve $csr; done
```

Then if there are Pending ones, you should see an output similar to below:
~~~bash
certificatesigningrequest.certificates.k8s.io/csr-26rcg approved
certificatesigningrequest.certificates.k8s.io/csr-4k6n8 approved
(...)
~~~

> **NOTE**: If you needed to do this, it may take a few minutes for the worker to be in a `Ready` state, this is due to it needing to deploy all of the necessary pods. We can proceed though and it'll catch up in the background.

Next let's validate the version that we've got deployed, and the status of the cluster operators:

```execute
oc get clusterversion
```

Then you should see :

~~~bash
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.9.5     True        False         27m     Cluster version is 4.9.5.
~~~

After that check the cluster operators 
=======

```execute
oc get clusteroperators
```

This command will list the all cluster operators and their availability as below

~~~bash
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.9.5     True        False         False      7h26m   
baremetal                                  4.9.5     True        False         False      7h48m   
cloud-controller-manager                   4.9.5     True        False         False      7h51m   
cloud-credential                           4.9.5     True        False         False      8h      
cluster-autoscaler                         4.9.5     True        False         False      7h48m   
config-operator                            4.9.5     True        False         False      7h50m   
console                                    4.9.5     True        False         False      7h29m   
csi-snapshot-controller                    4.9.5     True        False         False      7h48m   
dns                                        4.9.5     True        False         False      7h48m   
etcd                                       4.9.5     True        False         False      7h48m   
image-registry                             4.9.5     True        False         False      7h18m
(...)
~~~

### Making sure OpenShift is fully functional

OK, so this is likely something that you've done before, but in an attempt to validate OpenShift is ready for our lab, let's have a little bit of fun. Let's build a simple web-browser based game (called Duckhunt) from source, expose it via a route, and make sure all of the networking is hooked up properly. We'll use the **s2i** (source to image) container type:

Start with creating a new project with following command:

```execute
 oc new-project test
```
Now execute following command to deploy example application:

```execute
oc new-app \
	nodejs~https://github.com/vrutkovs/DuckHunt-JS
```

This will create all Kubernetes resources to deploy and run the example application as below:

~~~bash
--> Creating resources ...
    imagestream.image.openshift.io "duckhunt-js" created
    buildconfig.build.openshift.io "duckhunt-js" created
    deployment.apps "duckhunt-js" created
    service "duckhunt-js" created
--> Success
    Build scheduled, use 'oc logs -f buildconfig/duckhunt-js' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/duckhunt-js'
    Run 'oc status' to view your app.
~~~


Our application will now build from source, you can watch it happen by tailing the build log file. When it's finished it will push the image into the OpenShift image registry:

```execute
oc logs duckhunt-js-1-build -f
```

And you should wait for an output to see new image is pushed to registry:

~~~bash
Successfully pushed image-registry.openshift-image-registry.svc:5000/test/duckhunt-js@sha256:c4e64bc633ae09ce0f2f2f6de2ca9eaca8e11dc5b335301a2be78216df4b6929
Push successful
~~~

> **NOTE**: You may get an error saying "Error from server (BadRequest): container "sti-build" in pod "duckhunt-js-1-build" is waiting to start: PodInitializing"; you were just too quick to ask for the log output of the pods, simply re-run the command.

To check status of the pods, execute below command:

```execute
oc get pods 
```

You'll see that a couple of pods have been created, one that just completed our build, and then the application itself, which should be in a `Running` state, if it's still showing as `ContainerCreating` just give it a few more seconds:


~~~bash
NAME                           READY   STATUS      RESTARTS   AGE
duckhunt-js-1-build            0/1     Completed   0          4m7s
duckhunt-js-5b75fd5ccf-j7lqj   1/1     Running     0          105s   <-- this is our app!
~~~

Now expose the application (via the service) so we can route to it from the outside...


```execute
oc expose svc/duckhunt-js
```

As a result, a route is created:

~~~bash
route.route.openshift.io/duckhunt-js exposed
~~~

To check the route execute following command:

```execute
oc get route duckhunt-js
```

Now you should be able to see the route endpoint as below:

~~~bash
NAME          HOST/PORT                                  PATH   SERVICES      PORT       TERMINATION   WILDCARD
duckhunt-js   duckhunt-js-test.apps.%cluster_subdomain%          duckhunt-js   8080-tcp                 None
~~~

You should be able to open up the application in the same browser that you're reading this guide from, either copy and paste the address, or click this link: [http://duckhunt-js-test.%cluster_subdomain%](http://duckhunt-js-test.%cluster_subdomain%). If your OpenShift cluster is working as expected and the application build was successful, you should now be able to have a quick play with this... good luck ;-)
> **NOTE**: If you've deployed this environment via RHPDS, your URL above may be slightly different, and the hyperlink above will not work as expected. Use the output from the `oc get route duckhunt-js` as the correct route/address to use.

<img src="img/duckhunt.png"/>

Before we start looking at OpenShift Virtualization, let's just clean up the test project and have OpenShift remove the resources...

```execute
oc delete project test
```
Then wait for project deletion

~~~bash
project.project.openshift.io "test" deleted
~~~
